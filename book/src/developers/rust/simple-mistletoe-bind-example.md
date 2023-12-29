# Simple `mistletoe-bind` example

To make creating packages in Rust a little easier, we provide a `mistletoe-bind` crate that contains macros and libraries to provide higher-level abstractions of the underlying WebAssembly bindings.

On the [Overview](../../overview.html) page we saw a quick Rust example:

```rust
use indoc::formatdoc;
use mistletoe_api::v0_1::{MistHuskResult, MistHuskOutput};
use mistletoe_bind::misthusk_headers;
use serde::Deserialize;

misthusk_headers! {"
  name: example-namespace
  labels:
    mistletoe.dev/group: mistletoe-examples
"}

#[derive(Deserialize)]
struct InputConfig {
    name: String,
}

fn generate(input_config: InputConfig) -> MistHuskResult {
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

But that's just the body of the file.  Let's catch up to what the full project looks like.  The structure only has two files:
```
/src/lib.rs
/Cargo.toml
```

The contents of `Cargo.toml` is:

```toml
[package]
name = "mistletoe-example-basic-namespace"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
indoc = "2.0"
mistletoe-api = "0.1"
mistletoe-bind = "0.1"
serde = { version = "1.0", features = ["derive"] }
wasm-bindgen = "0.2"
```

For a base package, everything here is required aside from `indoc`, which we added to leverage the `formatdoc!{}` macro in the above.

## Breaking it down

Let's go top-to-bottom on the Rust source code and cover what each portion does.  To start, there's a macro that generates headers for our package:

```rust
misthusk_headers! {"
  name: example-namespace
  labels:
    mistletoe.dev/group: mistletoe-examples
"}
```

The input of this macro is a YAML string eerily reminiscent of the "metadata" section of Kubernetes resources.  The only required field here is the `name` of the package.  You can also add labels containing additional information about the package.  For now, you may want to add a `mistletoe.dev/group` label with the name of your project or organization.

This generates some binding functions that we'll cover [later in the section](./simple-mistletoe-bind-example.html#the-generated-code-from-misthusk_headers), but for now the only thing we need to worry about is that it will look for a `generate` function defined by you.

Let's look at ours, as well as the input object we defined for it:

```rust
#[derive(Deserialize)]
struct InputConfig {
    name: String,
}

fn generate(input_config: InputConfig) -> MistHuskResult {
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

The important part here is the signature: `(input_config: InputConfig) -> MistHuskResult`

Essentially, you can specify a function that takes any parameter that implements `Deserialize` (and that includes `String`), and outputs our `MistHuskResult` object.

The input we'll be receiving is an arbitrary YAML document, with at least `name` specified in it (and potentially more fields in the future).  For example, maybe the user will pass you these values:

```yaml
name: my-namespace
labels:
  app: my-app
  app.kubernetes.io/name: my-app
  app.kubernetes.io/instance: my-installation
```

To accommodate, you might expand your struct definition to:

```rust
#[derive(Deserialize)]
struct InputConfig {
    name: String,
    #[serde(default)] // If we don't receive it, just represent it here as `None`
    labels: Option<BTreeMap<String, String>>,
}
```

And it would just work as you'd expect!

### MistHuskResult

Before we get sidetracked, let's look at the contents of that function from before:

```rust
fn generate(input_config: InputConfig) -> MistHuskResult {
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

Aside from the handy `formatdoc!{}` macro from the [`indoc` crate](https://crates.io/crates/indoc), the only unexplained types here are `MistHuskResult` and `MistHuskOutput`.  To start, let's look at the definition of `MistHuskResult`:

```rust
type MistHuskResult = anyhow::Result<MistHuskOutput>
```

So it's really just a wrapper around `MistHuskOutput`.  The error message is populated up to the user, so here is where you would put end-user input.  For a quick example, let's create an arbitrary `anyhow` error, with a failure message:

```rust
return Err(anyhow!("dumb failure"));
```

The package run would exit unsuccessfully, and send the error message `"dumb failure"` on up to the user.  Because it leverages the [`anyhow` crate](https://crates.io/crates/anyhow), this also works with any other error via the `?` operator:

```rust
let parsed_yaml = serde_yaml::from_str(some_input_str)?;

// Since it's `anyhow`, we can also attach arbitrary failure messages
// to errors that we didn't make.
let parsed_yaml = serde_yaml::from_str(some_input_str)
    .with_context(|| format!("failed to parse some inner yaml"))?;
```

Going over the other part of the type, we see that the actual content of it is `MistHuskOutput`, which is the object we constructed at the start of the function.

### MistHuskOutput

*(TODO: continue this and talk about the `MistHuskOutput` object.)*

## Putting it in a package

*(TODO)*

## The generated code from `misthusk_headers`

*(TODO)*
