+++
title = 'Introducing Mistletoe (WIP)'
description = 'the Polyglot Package Manager for Kubernetes'
data = '2023-12-28T18:21:00-05:00'
+++

**Hello world!**  There's a [quick summary and roadmap](/) on the main page of the site, but I want to get into the nitty-gritty of **why**, **how**, and **when** the project is landing.  Things are moving *very* fast, and the project has gone from nothing to one-quarter-of-a-package-manager in under a week.

For a quick recap, **Mistletoe** is a WebAssembly-based package manager for Kubernetes.  The packages are WebAssembly modules with a function that takes an input YAML string and returns a Kubernetes YAML string.  The goal is to provide as much freedom as possible when writing packages.

## Why?

Right out the gate, I want to make something clear: [Helm](https://helm.sh/) is an amazing project and ecosystem.  I regularly use it and will continue to use it, especially during the early days of **Mistletoe.**

I also want to set expectations and make another thing clear: I'm not sure how long the early days of **Mistletoe** will go on for.  The base design is done, and I'm confident it's gone beyond the proof-of-concept phase, but there are so many rough edges and missing features that I can't recommend its use yet, and I also don't know when I can.

Ultimately the **why** of the situation has nothing to do with using Helm as an end-user.  It's near-perfect: I pick a chart, provide some input configuration, and it deploys the application to my cluster.  It's not one-size-fits-all, but it's pretty damn close.

Where I've had trouble is in the *writing packages.*  Helm packages are Golang templates of Kubernetes manifests, and the workflow pretty readily accommodates trivial-to-intermediate deployments.  Where it falls apart is on *advanced deployments.*

### An advanced deployment example

To explain that part, let's go beyond the deployment of individual applications, and let's talk about *a full deployment of a product platform to Kubernetes.*

Here's the gist: we have three microservices, `mic-a`, `mic-b`, and `mic-c`, and they have the following properties:

1. `a` and `b` share a MariaDB instance, `b` and `c` share a Redis instance.
2. `a`, `b`, and `c` all need to be aware of each other.

We're assuming there are already packages for each microservice, as well as one for MariaDB and one for Redis.

For example, here's what the MariaDB package might provide:

```rust
#[derive(Clone, Serialize, Deserialize)]
pub struct MariaDbInputs {
    pub name: String,
    pub namespace: String,
    pub users: Vec<MariaDbUser>,
}

#[derive(Clone, Serialize, Deserialize)]
pub struct MariaDbUser {
    pub username: String,
    pub password: String,
}

#[derive(Clone, Serialize, Deserialize)]
pub struct MariaDbExports {
    pub fqdn: String,
}

impl MariaDbInputs {
    pub fn new(name: &str, namespace: &str, users: Vec<&MariaDbUser>) -> Self {
        /* ... */
    }
}

impl MariaDbUser {
    /// Creates a user with a secure, random password
    pub fn create_secure(username: String) -> MariaDbUser { /* ... */ }
}

/// This is the function that every Mistletoe package provides, and is both
/// an entrypoint for direct execution via `mistctl`, as well as for other
/// packages to call.
pub fn generate(inputs: MariaDbInputs) -> MistHuskResult {
    /* ...return the YAML for a MariaDB Deployment... */
}
```

...and we can assume the Redis package will be similar.

Now let's go over what the code for our enclosing platform package would ideally look like in **Mistletoe**:

```rust
#[derive(Clone, Serialize, Deserialize)]
pub struct MyPlatformInputs {
    pub name: String,
    pub namespace: String,
}

struct MyMicroserviceInputs { /* ...all microservice inputs... */ }
impl Into<AInputs> for &MyMicroserviceInputs { /* ...just the mic-a inputs... */ }
impl Into<BInputs> for &MyMicroserviceInputs { /* ...just the mic-b inputs... */ }
impl Into<CInputs> for &MyMicroserviceInputs { /* ...just the mic-c inputs... */ }

pub fn generate(inputs: MyPlatformInputs) -> MistHuskResult {
    // 1. Create some user definitions
    let mariadb_user_a = MariaDbUser::create_secure("mic_a");
    let mariadb_user_b = MariaDbUser::create_secure("mic_b");
    let redis_user_b = RedisUser::create_secure("mic_b");
    let redis_user_c = RedisUser::create_secure("mic_c");

    // 2. Create the MariaDB and Redis instances
    let mariadb_instance = mariadb_mist::generate(MariaDbInputs::new(
        format!("{}-mariadb", input.name),
        &input.namespace,
        vec![&mariadb_user_a, &mariadb_user_b],
    ))?;
    let redis_instance = redis_mist::generate(RedisInputs::new(
        format!("{}-redis", input.name),
        &input.namespace,
        vec![&redis_user_b, &redis_user_c],
    ))?;

    // 3. Dump all the configuration that our microservices will want into a struct
    let child_inputs = MyMicroserviceInputs {
        name: &inputs.name,
        namespace: &inputs.namespace,

        mic_a_name: format!("{}-mic-a", &inputs.name),
        mic_b_name: format!("{}-mic-b", &inputs.name),
        mic_c_name: format!("{}-mic-c", &inputs.name),

        mariadb_instance: &mariadb_instance, mariadb_user_a, mariadb_user_b,
        redis_instance: &redis_instance, redis_user_b, redis_user_c,
    }

    // 4. Generate and attach all the YAML to our package's output
    let output = MistHuskOutput::new()
        .with_files_from(&mariadb_instance)
        .with_files_from(&redis_instance)
        .with_files_from(&mic_a_mist::generate(&child_inputs.into())?)
        .with_files_from(&mic_b_mist::generate(&child_inputs.into())?)
        .with_files_from(&mic_c_mist::generate(&child_inputs.into())?);

    // 5. Success!
    Ok(output)
}
```

> *Heavy disclaimer: the above code is **make-believe code** and the libraries to support it aren't complete yet -- but they are getting pretty close.*

There we go!  That's a package that wraps a whole platform stack, and is contained in a way that you can deploy several instances of it to the same cluster.

The fact that packages can share APIs allows for workflows that would otherwise be impossible in a Helm chart (unless you're willing to **really** abuse the template lang).

Creating and sharing the user objects with secure passwords is one of these workflows.  To get it to work in Helm, you have a couple paths: one is that you pre-generate the passwords and put it into the input YAML values for your dependencies.  This is less secure and puts it outside of the control of the parent package.

The other is to use the global namespace, at which point you have to:

1. Put in hacks to make sure that your template loads first,
2. Put in some front matter to generate the password and declare it *globally* (re: to all dependencies), and
3. *Make sure the dependencies use the global value.*  This means that you probably **can't** do this kind of workflow for packages you didn't write, like the MariaDB and Redis packages.

I want to thoroughly solve all these problems, and a maximalist position on developer freedom seems like the way to go.

## How?

The **how** of it all has been surprisingly easy.  Since I'm not writing a full engine, just using a hosted [Wasmer](https://wasmer.io/) runtime, I don't really need to lift my fingers much on creating a package runner.

The main work is going to be providing standards and an API surface that developers would actually want to use.  The goal is to make writing packages as easy, if not easier, than Helm.

To that end, I'm going to work on providing a simple and stable Rust interface, as well as go a bit further and support template engines, languages, and libraries with similar goals.

For instance, maybe you want to skip Rust and just write Handlebars templates, or maybe you want to leverage something like [cdk8s](https://cdk8s.io/) in Go or TypeScript to generate the YAML.

I also want to mention up front an idea I have for **Mistletoe** way down the line: *additional engines:*

* Helm is written in Go.
* Kustomize is written in Go.
* Go can compile to WebAssembly.

I hope you see where I'm going with this -- if I can get those two hosted into the toolset, we could theoretically have interop with both.  Though I have no clue the level of effort involved in doing this, just that I'm going to give a serious attempt at it if **Mistletoe** gains momentum.

## When?

So far the answer is "it depends on what you want".

I'm going to continue on stabilizing and expanding the core loop so that writing packages is a breeze.  At some point, hopefully in the short-to-medium-term future, it will be somewhat stable and ready for developers to use for packaging their own services.

The hard part is providing a desirable ecosystem for end-users.  Like any new ecosystem, it requires support and momentum, and the only thing I can do to that end is to provide as complete a picture as I can on my own, and hope that it eventually reaches a critical mass.

## So what can I do?

For now, keep reading the blog and follow along with development!  I'm also lobbying for comments, ideas, and constructive criticism to give it the best life possible.  Definitely feel free to reach out to [greg@fraley.dev](mailto:greg@fraley.dev) with your thoughts!

It's still small and fast-moving enough that pulling in additional help could result in a "too many cooks" situation.  And I can't in good faith advocate use yet.  But things are moving *fast,* and I'm excited for the day that I can get other people involved and advertise this as a batteries-included, top-to-bottom solution.
