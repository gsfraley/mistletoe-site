# Mistletoe for users

This section is still very in-flux, mainly because the tool itself is very in-flux.  The following may be subject to abrupt change, but we'll keep this book in sync of any changes.

As of now, there are really just two useful commands, `inspect` and `generate`:

```txt
> mistctl --help
Polyglot Kubernetes package manager

Usage: mistctl [COMMAND]

Commands:
  generate  Generate output YAML from a package
  inspect   Inspects the info exported by a package
  help      Print this message or the help of the given subcommand(s)

Options:
  -h, --help  Print help
```

# `inspect`

`inspect` calls the info function from the **Mistletoe** package and then exits:

```txt
> mistctl inspect --help
Inspects the info exported by a package

Usage: mistctl inspect [package]

Arguments:
  [package]  the package to inspect

Options:
  -h, --help  Print help
```

For now, it currently surfaces the raw YAML MistPackage value:

```txt
> mistctl inspect mistletoe/examples/namespace-example:0.1.1
apiVersion: mistletoe.dev/v1alpha1
kind: MistPackage
metadata:
  name: namespace-example
  labels:
    mistletoe.dev/group: mistletoe-examples
```

# `generate`

This function actually calls the package and generates the output YAML -- note that this does not install it on your cluster.

```txt
> mistctl generate --help
Generate output YAML from a package

Usage: mistctl generate [OPTIONS] --package <PACKAGE> <name>

Arguments:
  <name>  the name of the installation

Options:
  -p, --package <PACKAGE>  package to call
  -f, --inputfile <FILE>   input file containing values to pass to the package
  -s, --set <VALUES>       set values to pass to the package
  -o, --output <TYPE>      output type, can be 'yaml', 'raw', or 'dir=<dirpath>'
  -h, --help               Print hel
```

The only required parameters are the `name` and `--package`.  For instance, this is all our "namespace-example" package needs:

```txt
> mistctl generate my-namespace -p mistletoe/examples/namespace-example:0.1.1
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

Additional input can be passed in with `--inputfile` and `--set`.  `inputfile` is just a path to a YAML file of inputs you wish to pass into the package:

```txt
> cat inputs.yaml
namespace: my-namespace
// ...

> mistctl generate my-nginx -p mistletoe/fake-nginx-example -f inputs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: my-namespace
// ...
```

You can also control the output format with `--output`.  This can be either `raw`, `yaml`, or `dir=<output_dir>`, with `yaml` being the default.

`raw` outputs the raw response from the package:

```txt
> mistctl generate my-namespace -p mistletoe/examples/namespace-example:0.1.1 --output raw
apiVersion: mistletoe.dev/v1alpha1
kind: MistResult
data:
  result: Ok
  files:
    namespace.yaml: |
      apiVersion: v1
      kind: Namespace
      metadata:
        name: my-namespace
```

`yaml` outputs the processed contents of the files:

```txt
> mistctl generate my-namespace -p mistletoe/examples/namespace-example:0.1.1 --output yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

And `dir` outputs the contents of the file into the filetree as specified by the package:

```txt
> mistctl generate my-namespace -p mistletoe/examples/namespace-example:0.1.1 --output dir=out
> find ./out
./namespace.yaml 
```
