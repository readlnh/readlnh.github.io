---
title: Modules
date: 2020-6-20 19:32
tags: 
    - Rust
    - Modules
---

A module is a colletction of items: 
- functions
- structs
- traits
- `impl` blocks
- other modules

## features
Split code in logical units(modules), manage visibility(public/private) between them.

## Visibility
By **default**, the items in a module have **private visibility**, but this can be **overridden** with `pub`. Only the public items of a module can be accessed from outside the module scope.

```rust
mod my_mod {
    // Items in modules default to private visibility.
    fn private_function() {
        println!("called `my_mod::private_function()`");
    }

    // Use the `pub` modifier to override default visibility.
    pub fn function() {
        println!("called `my_mod::function()`");
    }

    // Items can access other items in the same module,
    // even when private.
    pub fn indirect_access() {  
        print!("called `my_mod::indirect_access()`, that\n> ");
        private_function();
    }

    pub mod nested {
        pub fn function() {
            println!("called `my_mod::nested::function()`");
        }        
    }
}
```

Public items, including those inside nested modules, can be accessed from outside the parent module.

```rust
fn main() {
    my_mod::function();
    my_mod::indirect_access();
    my_mod::nested::function();
}
```

### `pub(crate)`

Functions declared using `pub(crate)` syntax are only visible within the current crate.

```rust
mod my_mod {
    pub(crate) fn public_function_in_crate() {
        ...
    }
}

fn main() {
    ...
    my_mod::public_function_in_crate();
}
```

`pub(crate)` can be called from anywhere in the same module.

### `pub(in path)`

Functions declared by `pub(in path)` syntax are only visible within the given path. `path` must be a **parent or ancestor** module.

```rust
mod my_mod {
    ...
    pub mod nested {
        ...
        pub(in crate::my_mod) fn public_function_in_my_mod() {
            print!("called `my_mod::nested::public_function_in_my_mod()`, that\n> ");
        } 
    }
}
```

`pub(crate)` items can only be called from within the module specified.

```rust
fn main() {
    ...
    // Error `public_function_in_my_mod()` is private.
    // my_mod::nested::public_function_in_my_mod();
}
```

### `pub(self)`
Functions declared by `pub(self)` syntax are only visible within the **current module**, which is **same as leaving them private.**

### `pub(super)`
Functions declared using `pub(super)` syntax are only visible within the **parent module**.

### private module
Nested modules follow the same rules for visibility.

```rust
my_mod {
    ...
    // Nested modules follow the same rules for visibility
    mod private_nested {
        #[allow(dead_code)]
        pub fn function() {
            println!("called `my_mod::private_nested::function()`");
        }

        // Private parent items will still restrict the visibility of a child item,
        // even if it is declared as visible within a bigger scope.
        #[allow(dead_code)]
        pub(crate) fn restricted_function() {
            println!("called `my_mod::private_nested::restricted_function()`");
        }
    }
}
```

## struct visibility
Structs have an extra level of visibility with their fields. The visibility **defaults to private**, and can be overridden with `pub` modifier.

## `use` and `as`
The `use` declaration can be used to bind a full path to a new name, and the `as` keywords can bind imports to a different name.

```rust
use xxx::yyy::function1;
use xxx::yyy::function2 as f2;

fn main() {
    function1();
    f2();
}
```

## `super` and `self`

The `super` keyword refers to the **parent scope**, the `self` keyword refers to the current module scope - in the following case `mod1`. Note we can use `self` to access another module inside `mod1`(in this case `mod2`).

```rust
fn f() {
    println!("called `f()`");
}

mod mod1 {
    pub fn test_super() {
        super::f();
    }

    fn test_self() {
        println!("called `test_self()`");
    }

    mod mod2 {
        pub fn function_in_mod2() {
            println!("called `function_in_mod2()`");
        }
    }

    pub fn test_all() {
        test_super();
        test_self();
        self::test_self();
        self::mod2::function_in_mod2();
    }
}

fn main() {
    mod1::test_all();
}
```

## File heirarchy
Modules can be mapped to a file/directory.

For example:

```
.
├── main.rs
└── my_mod
    ├── mod.rs
    └── nested_mod.rs
```

*main.rs:*

```rust
mod my_mod;

fn main() {
    my_mod::function();
    my_mod::nested_mod::function();
    println!("Hello, world!");
}
```

Note the `mod my_mod`, this declaration will look for a file named `my_mod.rs` or `my_mod/mod.rs` and will insert its contents inside a module named `my_mod` under this scope.

*mod.rs:*

```rust
pub mod nested_mod;

pub fn function() {
    println!("called `my_mod::function()`");
}
```

Same as before. `pub mod nested_mod;` will locate the `nested_mod.rs` and insert it here under their respective modules. Note it follows the same rules for visibility. For instance, if we remove `pub` from `pub mod nested_mod`, we couldn't call `my_mod::nested_mod::function();` in `main.rs`.