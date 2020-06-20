# How to use local unpublished crate in Rust by cargo
Cargo is configured to look for dependencies on [crates.io](https://crates.io/) by defalut. However I want to use a local crate. Fortunately, cargo supports to use libraries ont only on  [crates.io](https://crates.io/), but also ther registries, `git` repositories or subdirectories on our local file system.

## create a `lib`
Firstly, create a new package:
```bash
$ cargo new test_crate --lib
```

Here we pass `--lib` because we're making a library. Then the cargo generates:

```
.
├── Cargo.lock
├── Cargo.toml
└── src
    └── lib.rs

1 directory, 3 files

```

Let's do something in `lib.rs`.

```rust
pub fn public_function_in_test_crate() {
    println!("called `public_function_in_test_crate`");
}

pub mod cool {
    pub fn cool_function() {
        println!("called `cool::cool_function`");
    }
}
```

## set `Cargo.toml`
Then we create a new binary program use `cargo new test_extern_crate` and add a dependency section to our executable's `Cargo.toml` and sepcify the path:

```toml
[dependencies]
test_crate = {path = "../test_crate"}
```

or

```toml
[dependencies.test_crate]
path = "../test_crate"
```

Now we can use our local crate `test_crate` as folliwing:

*main.rs:*

```rust
extern crate test_crate;

use test_crate::cool;

fn main() {
    test_crate::public_function_in_test_crate();
    test_crate::cool::cool_function();
    cool::cool_function();
}
```

more details in more detals in [Cargo book](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) .