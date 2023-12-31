# Mistletoe for developers

The default (and currently sole) engine in Mistletoe is the WebAssembly runtime.  While the [WebAssembly interface](./developers/webassembly-interface.md) is pretty simple, I first want to get into the higher-level aspects of it.

Specifically, Mistletoe uses the WebAssembly interface to pass around CRDs.  Let's follow the full run cycle of a package to cover our bases here.  At a high level, it looks like this:

1. **Mistletoe** calls the "info" method inside the package to grab information about it and how to run it.
2. Then **Mistletoe** call the "generate" method with an input string.
3. The package returns an output string, **Mistletoe** does some post-processing on it, and sends it into the next step of the process.

It's pretty simple, and made simpler by the fact that the strings we're passing around are actually Kubernetes CRDs, so they should be a familiar format to anyone in the Kubernetes ecosystem.  Let's see each one.

## MistPackage

This is that "info" method I was talking about.  It takes *no* parameters and returns something like the following YAML:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistPackage
metadata:
  name: nginx-example
  labels:
    mistletoe.dev/group: mistletoe-examples
```

Right now, it's just some metadata, aka a name and some optional labels.  The labels can be whatever you want, but for now there is one special one, and that's `mistletoe.dev/group`.  This could be the name of your organization or the name of a multi-package project -- it's mainly used for identifying it to the user.  But notably, it's still optional.

There may be future versions of the resource with more fields containing configuration information, but for now you only need to consider the above.

## MistInput

And then there's MistInput.  This is what gets passed into your packages "generate" function, and can be practically anything:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistInput
data:
  name: my-nginx
  namespace: my-namespace
```

It only has a "data" field, and it's almost completely freeform.  It's mostly a passthrough for the inputs passed in by the user.  For instance, if you later decide that you want your users passing in a port and maybe some labels, you might write your application to accept the following input:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistInput
data:
  name: my-nginx
  namespace: my-namespace
  port: 3001
  labels:
    extra-labels: with-extra-values
```

Your user can pass in this input like so:

```sh
mistctl generate my-nginx -p ./my-nginx-package.mist-pack.wasm -f input.yaml
```

Where "input.yaml" might look like this:

```yaml
namespace: my-namespace
port: 3001
labels:
  extra-labels: with-extra-values
```

Note that the "name" isn't specified here.  That's because it's a *special parameter* -- the engine passes in the value of the installation name to that field.

It's strongly recommended that you **don't** fail on unexpected fields, because the engine or maybe even your package may expect more fields down the line.

## MistResult

This is the string that your package returns, and it has two modes: `Ok` and `Err`.  Let's see `Err` first:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistResult
data:
  result: Err
  message: 'something went wrong'
```

It's pretty straight-forward, and only contains two fields: "result" and "message".  The "result" identifies this as an `Err`, and the required "message" is passed back to the end-user.  If needed, the "message" can be multiple lines.

Let's be more optimistic and assume it ran successfully.  Let's look at the output of our namespace example on the overview page:

```yaml
apiVersion: mistletoe.dev/v1alpha1
kind: MistResult
data:
  result: Ok
  message: 'nothing went wrong' # This line is optional
  files:
    namespace.yaml: |
      apiVersion: v1
      kind: Namespace
      metadata:
        name: my-namespace
```

The easy ones to explain are "result" and "message".  "result" must be set to `Ok`, and the "message" is still passed on up to the user, though note that the "message" is now *optional* given that we're succeeding.

The important part is the "files".  This is a map from filename to multi-line string, where the multi-line string is Kubernetes YAML that should be output to the file.

The filename is a relative path, and supports directories -- you can change the above name to `resources/namespace.yaml` and it would write the contents to `./out/resources/namespace.yaml` assuming your user passed in `-o dir=./out` to the calling command.

## What next?

And those are the only inputs and outputs at this point in time!  If you're looking to see *how* these strings are actually passed around, head on over to the [WebAssembly interface](./developers/webassembly-interface.md) section.  If you just want to get started on writing a package, check out the [Rust support](./developers/for-rust-developers.md) that's currently offered.
