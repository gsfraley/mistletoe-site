# The WebAssembly interface

The WebAssembly interface is pretty simple, and we've already covered the format of the strings getting passed around in [Mistletoe for developers](../for-developers.md).

All this section will cover is how these strings are written into and pulled out of the runtime.  So let's get to the gist and inspect a package:

```sh
wasmer inspect ./examples/namespace-example/pkg/mistletoe_namespace_example_bg.wasm
```

```yaml
Type: wasm
Size: 1.4 MB
Imports:
  Functions:
  Memories:
  Tables:
  Globals:
Exports:
  Functions:
    "__mistletoe_info": [] -> [I32]
    "__mistletoe_alloc": [I32] -> [I32]
    "__mistletoe_dealloc": [I32, I32] -> []
    "__mistletoe_generate": [I32, I32] -> [I32]
  Memories:
    "memory": not shared (18 pages..)
  Tables:
  Globals
```

So we see we only have four functions, and they all work in the standard pointer-sized type for WebAssembly, `I32`.  To start, I want to cover the `__mistletoe_info` function -- this is the "info" function from the previous section.  But specifically, I want to recover its return type, which is a pointer to a **fat pointer.**

A **fat pointer** is a two-pointer-sized array in the form of `[I32, I32]`.  The first `I32` is a pointer to the start of the return string.  The second `I32` is the length of the string.  All these pointers point around the `memory` exported by the package.

So when **Mistletoe** uses the "info" function, it performs the following steps:

1. It calls `__mistletoe_info` for *the pointer to our fat pointer.*  It then follows that pointer to a place in the `memory`.
2. It uses the fat pointer at the `memory` location by looking up the string by the pointer and length in the fat pointer.

## Allocation into the package

To ensure safe memory usage between the runtime and the package, we require that the package exports `__mistletoe_alloc` and `__mistletoe_dealloc` methods for the runtime to use cooperatively.

The workflow for this is pretty simple.  **Mistletoe** calls the "alloc" function with the length of the string it wishes to allocate, and the "alloc" function returns a pointer to an area in the `memory`.  **Mistletoe** then writes the string to the pointed location.

There's also the "dealloc" function that **Mistletoe** uses to clean up, which takes a pointer and length.  At the end of the run, this gets called twice -- first to deallocate any strings passed into the package, and then to dellocate anything returned by the package.

## The "generate" method

The real meat and potatoes is in the `__mistletoe_generate` function.  This is the main entrypoint into your package's logic.  To review, its signature looked like this:

```txt
[I32, I32] -> [I32]
```

The two parameters we pass in are the pointer and length of an input string we've allocated into the package.  What gets returned is a fat pointer to the output of the method.

**Mistletoe** copies the output out of the runtime, and deallocates everything sent into and retrieved out of the package.  And that was the last step -- the package has now run and the engine will process your output for use on the cluster.
