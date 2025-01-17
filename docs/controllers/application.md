# The Application

The **application** starts the [Controller] and links it up with the [[reconciler]] for your [[object]].

## Goal

This document shows the basics of creating a small controller with a `Pod` as the main [[object]].

## Requirements

We will assume that you have latest **stable** [rust] installed::

## Project Setup

```sh
cargo new --bin ctrl
cd ctrl
```

add then install `kube`, `k8s-openapi`, `thiserror`, `futures`, and `tokio`:

```sh
cargo add kube --features=runtime,client,derive
cargo add k8s-openapi --features=latest --default-features=false
cargo add thiserror
cargo add tokio --features=macros,rt-multi-thread
cargo add futures
```

<!-- do a content tabs feature here if it becomes free to let people tab between
This should give you a `[dependencies]` part in your `Cargo.toml` looking like:

```toml
kube = { version = "LATESTKUBE", features = ["runtime", "client", "derive"] }
k8s-openapi = { version = "LATESTK8SOPENAPI", features = ["latest"]}
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
futures = "0.3"
thiserror = "LATESTTHISERROR"
```
-->

This will populate some [`[dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) in your `Cargo.toml` file.

### Dependencies

- [kube] :: with the controller `runtime`, Kubernetes `client` and a `derive` macro for custom resources
- [k8s-openapi] :: structs for core Kubernetes resources at the `latest` supported [[kubernetes-version]]
- [thiserror] :: typed error handling
- [futures] :: async rust abstractions
- [tokio] :: supported runtime for async rust features

!!! warning "Alternate async runtimes"

    We depend on `tokio` for its `time`, `signal` and `sync` features, and while it is in theory possible to swap out a runtime, you would be sacrificing the most actively supported and most advanced runtime available. Avoid going down an alternate path unless you have a good reason.

Additional dependencies are useful, but we will go through these later as we add more features.

### Setting up errors

A full `Error` enum is the most versatile approach:

```rust
#[derive(thiserror::Error, Debug)]
pub enum Error {}

pub type Result<T, E = Error> = std::result::Result<T, E>;
```

### Define the object

Create or import the [[object]] that you want to control into your `main.rs`.

For the purposes of this demo we will import [Pod]:

```rust
use k8s_openapi::api::core::v1::Pod;
```

### Seting up the controller

This is where we will start defining our `main` and glue everything together:

```rust
#[tokio::main]
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let pods = Api::<Pod>::all(client);

    Controller::new(pods.clone(), Default::default())
        .run(reconcile, error_policy, Arc::new(()))
        .for_each(|_| futures::future::ready(()))
        .await;

    Ok(())
}
```

This creates a [Client], a Pod [Api] object (for all namespaces), and a [Controller] for the full list of pods defined by a default [watcher::Config].

We are not using [[relations]] here, this only schedules reconciliations when a pod changes.

### Creating the reconciler

You need to define at least a basic `reconcile` fn

```rust
async fn reconcile(obj: Arc<Pod>, ctx: Arc<()>) -> Result<Action> {
    println!("reconcile request: {}", obj.name_any());
    Ok(Action::requeue(Duration::from_secs(3600)))
}
```

and an error handler to decide what to do when `reconcile` returns an `Err`:

```rust
fn error_policy(_object: Arc<Pod>, _err: &Error, _ctx: Arc<()>) -> Action {
    Action::requeue(Duration::from_secs(5))
}
```

To make this reconciler useful, we can reuse the one created in the [[reconciler]] document, on a custom [[object]].

## Checkpoint

If you copy-pasted everything above, and fixed imports, you should have a `src/main.rs` in your `ctrl` directory with this:

```rust
use std::{sync::Arc, time::Duration};
use futures::StreamExt;
use k8s_openapi::api::core::v1::Pod;
use kube::{
    Api, Client, ResourceExt,
    runtime::controller::{Action, Controller}
};

#[derive(thiserror::Error, Debug)]
pub enum Error {}
pub type Result<T, E = Error> = std::result::Result<T, E>;

#[tokio::main]
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let pods = Api::<Pod>::all(client);

    Controller::new(pods.clone(), Default::default())
        .run(reconcile, error_policy, Arc::new(()))
        .for_each(|_| futures::future::ready(()))
        .await;

    Ok(())
}

async fn reconcile(obj: Arc<Pod>, ctx: Arc<()>) -> Result<Action> {
    println!("reconcile request: {}", obj.name_any());
    Ok(Action::requeue(Duration::from_secs(3600)))
}

fn error_policy(_object: Arc<Pod>, _err: &Error, _ctx: Arc<()>) -> Action {
    Action::requeue(Duration::from_secs(5))
}
```

## Developing

At this point, you are ready start the app and see if it works. I.e. you need Kubernetes.

### Prerequisites

> If you already have a cluster, skip this part.

We will develop locally against a `k3d` cluster (which requires `docker` and `kubectl`).

Install the [latest k3d release](https://k3d.io/#releases), then run:

```sh
k3d cluster create kube --servers 1 --agents 1 --registry-create kube
```

If you can run `kubectl get nodes` after this, you are good to go. See [k3d/quick-start](https://k3d.io/#quick-start) for help.

### Local Development

In your `ctrl` directory, you can now `cargo run` and check that you can successfully connect to your cluster.

You should see an output like the following:

```
reconcile request: helm-install-traefik-pxnnd
reconcile request: helm-install-traefik-crd-8z56p
reconcile request: traefik-97b44b794-wj5ql
reconcile request: svclb-traefik-5gmsm
reconcile request: coredns-7448499f4d-72rvq
reconcile request: metrics-server-86cbb8457f-8fct5
reconcile request: local-path-provisioner-5ff76fc89d-4x86w
reconcile request: svclb-traefik-q8zkw
```

I.e. you should get a reconcile request for every pod in your cluster (`kubectl get pods --all`).

If you now edit a pod (via `kubectl edit pod traefik-xxx` and make a change), or create a new pod, you should immediately get a reconcile request.

**Congratulations**. You have __technically__ built a kube controller.

!!! warning "Continuation"

    You have created the [[application]] using trivial reconcilers and objects, but you are not controlling anything yet. See the [[object]] and [[reconciler]] chapters for the business logic.

## Deploying

### Containerising

WIP. Showcase both multi-stage rust build and musl builds into distroless.

### Containerised Development

WIP. Showcase a basic `tilt` setup with `k3d`.

### Continuous Integration

See [[testing]] and [[security]].

## Advanced Topics
### Observability

See [[observability]] for common practices for metrics, traces and logs.

### Optimization and Stream Tuning
For techniques involving optimizing resource use, and fine tunining reconciler behaviour see:

- [[optimization]]
- [[streams]]

### Useful Dependencies

The following dependencies are **already used** transitively **within kube** that may be of use to you. Use of these will generally not inflate your total build times due to already being present in the tree:

- [tracing]
- [futures]
- [k8s-openapi]
- [serde]
- [serde_json]
- [serde_yaml]
- [tower]
- [tower-http]
- [hyper]
- [thiserror]

These in turn also pull in their own dependencies (and tls features, depending on your tls stack), consult [cargo-tree] for help minimizing your dependency tree.


--8<-- "includes/abbreviations.md"
--8<-- "includes/links.md"

[//begin]: # "Autogenerated link references for markdown compatibility"
[reconciler]: reconciler "The Reconciler"
[object]: object "The Object"
[kubernetes-version]: ../kubernetes-version "kubernetes-version"
[relations]: relations "Related Objects"
[testing]: testing "Testing"
[security]: security "Security"
[observability]: observability "Observability"
[optimization]: optimization "Optimization"
[streams]: streams "Streams"
[//end]: # "Autogenerated link references"
