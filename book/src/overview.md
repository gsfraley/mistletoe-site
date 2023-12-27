# Overview

**Mistletoe** is a package manager for Kubernetes.  In fact, usage is pretty similar to other package managers like [Helm](https://helm.sh/).

Where **Mistletoe** differs is the packages themselves -- instead of a templating engine like Helm or manifest processor like Kustomize, this package manager is built around a **WebAssembly** runtime, and a rather simple one at that.

Ultimately, almost all existing package management solutions run on a workflow of *"string-in-string-out"*.  They take some configuration string in YAML or another format, pass it into the package, and the package returns Kubernetes resource manifest strings.  The focus of **Mistletoe** is to open this workflow up to the maximum possible extent.

## What does it look like?

Let's take a quick look at what a package looks like and what it looks like when you go to run it.  To start, here's a very simple package written in Rust:

```rust
misthusk_headers! {"
  name: example-namespace
  labels:
    mistletoe.dev/group: mistletoe-examples
"}

#[derive(Deserialize)]
struct InputConfig {
    name: String,
}

pub fn generate(input_config: InputConfig) -> MistHuskResult {
    let output = MistHuskOutput::new()
        .with_file("namespace.yaml".to_string(), formatdoc!{"
            apiVersion: v1
            kind: Namespace
            metadata:
              name: {0}
        ", input_config.name});

    Ok(output)
}
```

We'll discuss and expand the above example in the [mistletoe-bind example section](./developers/rust/simple-mistletoe-bind-example.html), but for now we'll just talk about what it does.

It takes the `name` parameter that is passed in by the engine, and creates a Namespace with that name.  Generating the YAML output of this package is done with the `mistctl generate` command:

```sh
mistctl generate my-namespace ./example-namespace.mist
```

That will output:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

And that's the core workflow of **Mistletoe!**  We're just scratching the surface of what's possible, so read on in the chapters ahead for further usage, development guides, and hopefully inspiration for ideas.

## Disclaimer on the status of the project

Work has only just begun on Mistletoe.  While the core loop is done and necessary features are being rapidly added, this is NOT near recommended for use, not even on the cautious level of "beta".

That said: input, ideas, and constructive criticism are very welcome at the formative stages of this project -- I want to give it the best life possible.

More info on the roadmap can be found at the bottom of [the main site.](https://mistletoe.dev)
