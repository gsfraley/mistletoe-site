# For Rust developers

To make creating packages in Rust a little easier, we provide a `mistletoe-bind` crate that contains macros and libraries to provide higher-level abstractions of the underlying WebAssembly bindings.

On the [Overview](../../overview.html) page we saw a quick Rust example:

```rust
use mistletoe_api::v1alpha1::{MistResult, MistOutput};
use mistletoe_bind::mistletoe_package;

use indoc::formatdoc;
use serde::Deserialize;

mistletoe_package! {"
  name: namespace-example
  labels:
    mistletoe.dev/group: mistletoe-examples
"}

#[derive(Deserialize)]
pub struct Inputs {
    name: String,
}

pub fn generate(inputs: Inputs) -> MistResult {
    let name = inputs.name;

    let output = MistOutput::new()
        .with_file("namespace.yaml".to_string(), formatdoc!("
            apiVersion: v1
            kind: Namespace
            metadata:
              name: {name}
        "));

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
name = "mistletoe-namespace-example"
version = "0.1.2"
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

For a base package, everything here is required aside from `indoc`, which we added to leverage the `formatdoc!()` macro in the above.

## Breaking it down

Let's go top-to-bottom on the Rust source code and cover what each portion does.  To start, there's a macro that generates headers for our package:

```rust
mistletoe_package! {"
  name: namespace-example
  labels:
    mistletoe.dev/group: mistletoe-examples
"}
```

The input of this macro is actually the `metadata` section of the MistPackage object our package will return, covered in [Mistletoe for developers](../for-developers.md).  The only required field here is the `name` of the package.  You can also add labels containing additional information about the package.  For now, you may want to add a `mistletoe.dev/group` label with the name of your project or organization.

This generates some binding functions that we'll cover [later in the section](./for-rust-developers.html#the-generated-code-from-mistletoe_package), but for now the only thing we need to worry about is that it will look for a `generate` function defined by you.

Let's look at ours, as well as the input object we defined for it:

```rust
#[derive(Deserialize)]
pub struct Inputs {
    name: String,
}

pub fn generate(inputs: Inputs) -> MistResult {
    let name = inputs.name;

    let output = MistOutput::new()
        .with_file("namespace.yaml".to_string(), formatdoc!("
            apiVersion: v1
            kind: Namespace
            metadata:
              name: {name}
        "));

    Ok(output)
}
```

The important part here is the signature: `(inputs: Inputs) -> MistResult`

Essentially, you can specify a function that takes any parameter that implements `Deserialize` (and that includes `String`), and outputs our `MistResult` object.

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
pub struct Inputs {
    name: String,
    #[serde(default)] // If we don't receive it, just represent it here as `None`
    labels: Option<BTreeMap<String, String>>,
}
```

And it would just work as you'd expect!

### MistResult

Before we get sidetracked, let's look at the contents of that function from before:

```rust
pub fn generate(inputs: Inputs) -> MistResult {
    let name = inputs.name;

    let output = MistOutput::new()
        .with_file("namespace.yaml".to_string(), formatdoc!("
            apiVersion: v1
            kind: Namespace
            metadata:
              name: {name}
        "));

    Ok(output)
}
```

Aside from the handy `formatdoc!()` macro from the [`indoc` crate](https://crates.io/crates/indoc), the only unexplained types here are `MistResult` and `MistOutput`.  To start, let's look at the definition of `MistResult`:

```rust
type MistResult = anyhow::Result<MistOutput>
```

So it's really just a wrapper around `MistOutput`.  The error message is populated up to the user, so here is where you would put end-user input.  For a quick example, let's create an arbitrary `anyhow` error, with a failure message:

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

To circle back to the string interface mentioned in [Mistletoe for developers](../for-developers.md), the `MistResult` here serializes to the same MistResult mentioned there.  Specifically, the "Ok" case deserializes to a `result: Ok` MistResult, and likewise "Err" to `result: Err`.

### MistOutput

Going over the other part of the type, we see that the actual content of it is `MistOutput`, which is the object we constructed at the start of the function.  This is what's used to populate a "successful" MistResult, and it works like a builder.  Let's look at a more complex usage of it:

```rust
let output = MistOutput::new()
    .with_message("Things went well!")
    .with_file("namespaces/namespace1.yaml".to_string(), formatdoc!("
        apiVersion: v1
        kind: Namespace
        metadata:
            name: namespace1
    "))
    .with_file("namespaces/namespace2.yaml".to_string(), formatdoc!("
        apiVersion: v1
        kind: Namespace
        metadata:
            name: namespace2
    "));
```

If you notice, we can use it to add as many files as we want!  And we can organize them into a file tree as well.  The usage is simple, just give it a filename and the contents of the file.

We also set the message here.  This step is optional, but useful if you want to pass some information back to the user about the output of the package.  (I also don't recommend passing in the trivial "Things went well!" message, this is more for if there's something significant the user should know.)

All that's left is to package it up into our MistResult!

```rust
Ok(output)
```

## Putting it in a package

Ultimately, the rest of the work is taken care of by `wasm_bindgen` and `wasm-pack`.  To run the last steps, you'll need to install the latter tool to your system:

```sh
cargo install wasm-pack
```

And then it's just one command to spit out the module!  CD to your project directory and build a minimal version of it:

```sh
wasm-pack build --target no-modules
```

That should generate a `./pkg/<name>_bg.wasm` file -- this is your package, and you can run it directly by passing the path into the `--package` option of any `mistctl` command.

Optionally, you may want to rename it to the standard `<name>-<version>.mist-pack.wasm` filename convention.  This is particularly important if you want to put it in a registry, where this convention is required.

## The generated code from `mistletoe_package`

While this information isn't really needed for usage of the Rust libraries, this could be helpful if you wish to implement bindings in a different language or re-implement new bindings in Rust.

### The info function

Let's get start breaking down the content of the body of the `mistletoe_package` macro, starting with the "info" function:

```rust
const INFO: &'static str = #mistpackage_string;

static INFO_PTR: mistletoe_bind::include::once_cell::sync::Lazy<std::sync::atomic::AtomicPtr<[usize; 2]>>
    = mistletoe_bind::include::once_cell::sync::Lazy::new(||
{
    let wide_ptr = Box::new([INFO.as_ptr() as usize, INFO.len()]);
    std::sync::atomic::AtomicPtr::new(Box::into_raw(wide_ptr))
});

#[wasm_bindgen::prelude::wasm_bindgen]
pub fn __mistletoe_info() -> *mut [usize; 2] {
    unsafe { *INFO_PTR.as_ptr() }
}
```

The only template value being passed in here is `#mistpackage_string` value.  This is actually a wrapped version of the string you pass into it when calling it.  It takes this:

```rust
mistletoe_package! {"
  name: namespace-example
  labels:
    mistletoe.dev/group: mistletoe-examples
"}
```

And returns this:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistPackage
metadata:
  name: namespace-example
  labels:
    mistletoe.dev/group: mistletoe-examples
```

The end result of the "info" logic is that we embed that string into `const INFO`, create a static fat pointer to it at runtime in `static INFO_PTR`, and create a return function to send that fat pointer back to the engine.

### The allocation functions

The next two functions declared are for allocation, and they're pretty simple since they're not our logic:

```rust
#[wasm_bindgen::prelude::wasm_bindgen]
pub fn __mistletoe_alloc(len: usize) -> *mut u8 {
    unsafe {
        let layout = std::alloc::Layout::from_size_align(len, std::mem::align_of::<u8>()).unwrap();
        std::alloc::alloc(layout)
    }
}

#[wasm_bindgen::prelude::wasm_bindgen]
pub fn __mistletoe_dealloc(ptr: *mut u8, len: usize) {
    unsafe {
        let layout = std::alloc::Layout::from_size_align(len, std::mem::align_of::<u8>()).unwrap();
        std::alloc::dealloc(ptr, layout);
    }
}
```

We're really just exporting the Rust allocator API on up.

### The generate function

The last bit, and the bit with the most moving parts, is the generate function(s):

```rust
fn __mistletoe_generate_result(input_str: &str) -> mistletoe_api::v1alpha1::MistResult {
    let input: mistletoe_api::v1alpha1::MistInput = mistletoe_bind::include::serde_yaml::from_str(input_str)?;
    generate(input.try_into_data()?)
}

#[wasm_bindgen::prelude::wasm_bindgen]
pub fn __mistletoe_generate(ptr: *const u8, len: usize) -> *mut [usize; 2] {
    let input_str = unsafe { std::str::from_utf8(std::slice::from_raw_parts(ptr, len)).unwrap() };
    let result = __mistletoe_generate_result(input_str);
    let mut output_str = std::mem::ManuallyDrop::new(mistletoe_api::v1alpha1::serialize_result(result).unwrap());
    let retptr = Box::into_raw(Box::new([output_str.as_mut_ptr() as usize, output_str.len()]));
    retptr
}
```

So there's really two parts of the path to this function: going into the user-defined `generate` function, and going out.

Going into it, we take the pointer and length passed in to us from the engine and convert it into a string.  We then convert it into a `MistInput` object and then convert *that* into the user-defined type of their function.  In our example, our function was:

```rust
pub fn generate(inputs: Inputs) -> MistResult { /* ... */ }
```

It'll convert it into anything that implements `Deserialize` (including `String`), which in our case is our custom `Inputs` type.

Going out of it, we take the output of the `generate` function and serialize it to the standard MistResult string output.  We also wrap it to be manually-dropped, since we don't want to deallocate it before the runtime can read it -- the runtime will take care of that on our behalf.

Then we just create a fat pointer to it, create a pointer to the fat pointer, and send it on up to the runtime.  Our job is done!
