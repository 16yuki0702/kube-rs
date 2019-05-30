# kube-rs
[![Build Status](https://travis-ci.org/clux/kube-rs.svg?branch=master)](https://travis-ci.org/clux/kube-rs)
[![Client Capabilities](https://img.shields.io/badge/Kubernetes%20client-Silver-blue.svg?style=plastic&colorB=C0C0C0&colorA=306CE8)](http://bit.ly/kubernetes-client-capabilities-badge)
[![Client Support Level](https://img.shields.io/badge/kubernetes%20client-alpha-green.svg?style=plastic&colorA=306CE8)](http://bit.ly/kubernetes-client-support-badge)
[![Crates.io](https://img.shields.io/crates/v/kube.svg)](https://crates.io/crates/kube)

Rust client for [Kubernetes](http://kubernetes.io) in the style of [client-go](https://github.com/kubernetes/client-go). Contains rust reinterpretations of the `Reflector` and `Informer` abstractions (but without all the factories) to allow writing kubernetes controllers/operators easily.

This client caters to the more common controller/operator case, but allows you to compile with the `openapi` feature to get accurate struct representations via [k8s-openapi](https://github.com/Arnavion/k8s-openapi).

## Usage
See the [examples directory](./examples) for how to watch over resources in a simplistic way.

See [controller-rs](https://github.com/clux/controller-rs) for a full example with [actix](https://actix.rs/).

**[API Docs](https://clux.github.io/kube-rs/kube/)**

## Typed Api
It's recommended to compile with the "openapi" feature if you want an easy experience, and accurate native object structs:

```rust
let pods = OpenApi::v1Pod(client).within("default");

let p = pods.get("blog")?;
println!("Got blog pod with containers: {:?}", p.spec.containers);

let patch = json!({"spec": {
    "activeDeadlineSeconds": 5
}});
let patched = pods.patch("blog", &pp, serde_json::to_vec(&patch)?)?;
assert_eq!(patched.spec.active_deadline_seconds, Some(5));

pods.delete("blog", &DeleteParams::default())?;
```

See the `pod_openapi` or `crd_openapi` examples for more uses.

## Reflector
One of the main abstractions exposed from `kube::api` is `Reflector<P, U>`. This is a cache of a resource that's meant to "reflect the resource state in etcd".

It handles the api mechanics for watching kube resources, tracking resourceVersions, and using watch events; it builds and maintains an internal map.

To use it, you just feed in `T` as a `Spec` struct and `U` as a `Status` struct, which can be as complete or incomplete as you like. Here, using the complete structs via [k8s-openapi](https://docs.rs/k8s-openapi/0.4.0/k8s_openapi/api/core/v1/struct.PodSpec.html):

```rust
let api = Api::v1Pod().within(&namespace);
let rf : Reflector<PodSpec, PodStatus> = Reflector::new(client, api)
    .timeout(10)
    .init()?;
```

then you can `poll()` the reflector, and `read()` to get the current cached state:

```rust
rf.poll()?; // watches + updates state

// read state and use it:
rf.read()?.into_iter().for_each(|(name, p)| {
    println!("Found pod {} ({}) with {:?}",
        name,
        p.status.unwrap().phase.unwrap(),
        p.spec.containers.into_iter().map(|c| c.name).collect::<Vec<_>>(),
    );
});
```

The reflector itself is responsible for acquiring the write lock and update the state as long as you call `poll()` periodically.

## Informer
The other main abstraction from `kube::api` is `Informer<P, U>`. This is a struct with the internal behaviour for watching kube resources, but maintains only a queue of `WatchEvent` elements along with `resourceVersion`.

You tell it what type parameters correspond to; `T` should be a `Spec` struct, and `U` should be a `Status` struct. Again, these can be as complete or incomplete as you like. For instance, using the complete structs from [k8s-openapi](https://docs.rs/k8s-openapi/0.4.0/k8s_openapi/api/core/v1/struct.PodSpec.html):

```rust
let api = Api::v1Pod();
let inf : Informer<PodSpec, PodStatus> = Informer::new(client, api)
    .init()?;
```

The main feature of `Informer<P, U>` is that after calling `.poll()` you handle the events and decide what to do with them yourself:

```rust
inf.poll()?; // watches + queues events

while let Some(event) = inf.pop() {
    handle_event(&client, event)?;
}
```

How you handle them is up to you, you could build your own state, you can call a kube client, or you can simply print events. Here's a sketch of how such a handler would look:

```rust
fn handle_event(c: &APIClient, event: WatchEvent<PodSpec, PodStatus>) -> Result<(), failure::Error> {
    match event {
        WatchEvent::Added(o) => {
            let containers = o.spec.containers.into_iter().map(|c| c.name).collect::<Vec<_>>();
            println!("Added Pod: {} (containers={:?})", o.metadata.name, containers);
        },
        WatchEvent::Modified(o) => {
            let phase = o.status.phase.unwrap();
            println!("Modified Pod: {} (phase={})", o.metadata.name, phase);
        },
        WatchEvent::Deleted(o) => {
            println!("Deleted Pod: {}", o.metadata.name);
        },
        WatchEvent::Error(e) => {
            println!("Error event: {:?}", e);
        }
    }
    Ok(())
}
```

The [node_informer example](./examples/node_informer.rs) has an example of using api calls from within event handlers.

## Examples
Examples that show a little common flows. These all have logging of this library set up to `trace`:

```sh
# watch pod events in kube-system
cargo run --example pod_informer
# watch for broken nodes
cargo run --example node_informer
```

or for the reflectors:

```sh
cargo run --example pod_reflector
cargo run --example node_reflector
cargo run --example deployment_reflector
```

for one based on a CRD, you need to create the CRD first:

```sh
kubectl apply -f examples/foo.yaml
cargo run --example crd_reflector
```

then you can `kubectl apply -f crd-baz.yaml -n kube-system`, or `kubectl delete -f crd-baz.yaml -n kube-system`, or `kubectl edit foos baz -n kube-system` to verify that the events are being picked up.

## Timing
All watch calls have timeouts set to `10` seconds as a default (and kube always waits that long regardless of activity). If you like to hammer the API less, you can either call `.poll()` less often and the events will collect on the kube side (if you don't wait too long and get a Gone). You can configure the timeout with `.timeout(n)` on the `Informer` or `Reflector`.

## Raw Api
You can not use `k8s-openapi` if you only are working with Informers/Reflectors, or you are happy to supply partial definitions of the native objects you are working with. You will have to specify the complete expected output type to serialize as however:

```rust
#[derive(Deserialize, Serialize, Clone)]
pub struct FooSpec {
    name: String,
    info: String,
}
let foos = Api::customResource("foos")
    .version("v1")
    .group("clux.dev")
    .within("dev");

let fdata = json!({
    "apiVersion": "clux.dev/v1",
    "kind": "Foo",
    "metadata": { "name": "baz" },
    "spec": { "name": "baz", "info": "old baz" },
});
let req = foos.create(&PostParams::default(), serde_json::to_vec(&fdata)?)?;
let o = client.request::<FooMeta>(req)?;

let fbaz = client.request::<bject<FooSpec, Void>>(foos.get("baz")?)?;
assert_eq!(fbaz.spec.info, "old baz");
```

Most of the informer/reflector examples does this at the moment (but imports k8s_openapi manually to do it anyway). See the `crd_api` example for more info.

## License
Apache 2.0 licensed. See LICENSE for details.
