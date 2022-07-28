---
title: "Writing a Build Automation Tool"
date: 2022-07-08T15:20:06+05:30
draft: false
tags: ["rust", "automation", "GNU make"]
---

---

After using GNU Make for automating the build step in my projects, I had this idea of making my own build automation tool like Make.

I originally wanted to use Makefile-like syntax for the config file but instead settled with [TOML](https://toml.io) after thinking about it for a while.
I also used Rust to write this since I am kinda familiar with the language.

Rust has a [TOML crate](https://docs.rs/toml/latest/toml/) for parsing TOML files.
It also provides support for deserialization and serialization using [serde](https://docs.rs/serde/latest/serde/).

It's also extremely easy to use:

```rust
// we store custom and pre as HashMap<String, Struct> so that in the TOML file we can use dynamic table names.
// for environment variables, it's stored as HashMap<String, String> so we can specify as many env vars as we like.

#[derive(Debug, Deserialize)]
struct Recipe {
    build: Build,
    custom: Option<HashMap<String, Custom>>, // custom commands
    pre: Option<HashMap<String, Pre>>, // pre-build commands
    env: Option<HashMap<String, String>>, // for environment vars
}

/* <the build, custom and pre structs> */

let recipe: Recipe;

match toml::from_str(&recipe_str) {
    Ok(r) => recipe = r,
    Err(e) => {
        printb!("Error: {}", e);
        exit(1);
    }
}
```

I didn't include the other structs to make the snippet smaller. This was also the first time I used rust macros in a project, as you can see with the `printb!` macro. All it does is print
`Baker: <message>` as coloured output.

```rust
#[macro_export]
macro_rules! printb {
    ($($arg:tt)*) => {
        println!("\x1b[32mBaker:\x1b[0m {}", format!($($arg)*));
    };
}
```

### Configuring baker

The configuration file is put in the root directory of the project, just like the Makefile.
If Baker is unable to find a config file (called `recipe.toml`), it auto generates one.
For example, I use this config for my chess engine:

```toml
[build]
cmd = "cargo build && cp -r ./target/debug/cranium ./bin/cranium"

[custom.clean]
cmd = "cargo clean"
run = false

[custom.setup]
cmd = "mkdir -p bin && rustup install stable && rustup default stable"
run = false

[custom.release]
name = "release"
cmd = "cargo build --release && cp -r ./target/release/cranium ./bin/cranium"
run = false

[pre.fmt]
cmd = "cargo fmt"
```

Baker is invoked using `bake` and by default checks for the `build` field in the `recipe.toml`.
The `cmd` value is then executed during build.

You can also add custom fields using `custom`. It also checks for a `run` value that is used to check whether the command should be run
after build or not. So if `run = true`, then the command is run after build.

Baker also checks for an optional `pre` field that contains commands which should be run **before** build. Like for example, if you wish to
format your code or something before compiling.

You may have noticed a `name` value in the `custom` and `pre` fields (`[custom.name]`). This is used to identify each field, and for the `custom` field,
the `name` is set as an argument when Baker is invoked.

For example, if I have a `custom` field called `setup`:

```toml
[custom.setup]
cmd = "mkdir -p bin && rustup install stable && rustup default stable"
run = false
```

You can run `bake setup` to directly execute the command inside the field.

Baker is open source and you can find the repo [here](https://github.com/rv178/baker).
I wrote it in like 2 days, so it may have bugs which I was not able to find, so all suggestions are welcome!

That's it for this blog, see you soon!

---
