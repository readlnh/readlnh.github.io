---
title: Formatting
date: 2019-11-01 19:35
tags: 
    - Rust
---
Some notes with Rust by example about formatting.

## Formatted print
Printing is handle by a series of `macros` defined in `std::fmt` some of which include:

- `format!`: write formatted text to `String`
- `print!`: print fotmatted text to the console(io::stdout) 
- `println!`: same as `print!` but a newline is appended
- `eprint` : same as `format!`, but the text if printed to the santard error(io::stderr)
- `eprintln`: smae as `eprint`, but a newline is appended


`std::fmt` contains many `traits` which govern the display of text. The base form of two important ones are listed below:

- `fmt::Debug`: Uses the `{:?}` marker. Format text for debugging purpoes
- `fmt::Display`: Uses the `{}` marker. Format text in a more elegant, user friendly fashion.

Note: Implementing the `fmt::Display` trait automatically implements the `ToString` trait which allows us to `convert` the type to `String`.

## Debug 
All types which want to use `std::fmt` formatting `traits` require an implementation to be printable. Automatic implementations are only provided for types such as in the `std` library. All others must be* maually* implemented somehow.

All types can `derive`(automatically create) the `fmt::Debug` implementation. This is not true for `fmt::Display` which must be manually implemented.

```Rust
// cannot be printed either with `fmt::Display` or  with `fmt::Debug`
struct UnPrintable(i32);

// The `derive` attribute automatically creates the implementation 
// required to make this `struct` printable with `fmt::Debug` 
#[derive(Debug)]
struct DebugUnPrintable(i32);
```

`fmt::Debug` definitely makes this printable but scarifices some elegance.Rust also provides `pretty printing` with `{:#?}`

One can manually implemnet `fmt::Debug` instead of derive.

```Rust
use std::fmt::{self, Debug, Formatter};

struct TestDebug {
    v1: i32,
    v2: i32,
}

#[derive(Debug)]
struct TestDebugauto {
    v1: i32,
    v2: i32,
}

impl TestDebugauto {
    fn new(x: i32, y: i32) -> Self {
        TestDebugauto {
            v1: x,
            v2: y,
        }
    }
}

impl TestDebug {
    fn new(x: i32, y: i32) -> Self {
        TestDebug {
            v1: x,
            v2: y,
        }
    }
}

impl Debug for TestDebug {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(f, "TestDebug: {} {}", self.v1, self.v2)
    }
}

fn main() {
   let t = TestDebug::new(1, 2);
   let t2 = TestDebugauto::new(1, 2);
   println!("{:?}\n{:?}", t, t2);
}
```
*output*
```bash
   Compiling debug_manually v0.1.0 (/home/readlnh/workspace/rust_workspace/rust-by-example/debug_manually)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29s
     Running `target/debug/debug_manually`
TestDebug: 1 2
TestDebugauto { v1: 1, v2: 2 }
```

## Display
One can manually implement `fmt::Display` to control the display,same as `fmt::Debug`.

```Rust
use std::fmt;

struct TestDisplay {
    v1: i32,
    v2: i32,
}


impl TestDisplay {
    fn new(x: i32, y: i32) -> Self {
        TestDisplay {
            v1: x,
            v2: y,
        }
    }
}

impl fmt::Display for TestDisplay {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // Write strictly the two elements into the supplied output
        // stream: `f`. Returns `fmt::Result` which indicates whether the
        // operation succeeded or failed. Note that `write!` uses syntax which
        // is very similar to `println!`.
        write!(f, "TestDebug: {} {}", self.v1, self.v2)
    }
}

fn main() {
   let t = TestDisplay::new(1, 2);
   println!("{}", t);
}

```


*output*
```bash
   Compiling display_manually v0.1.0 (/home/readlnh/workspace/rust_workspace/rust-by-example/display_manually)
    Finished dev [unoptimized + debuginfo] target(s) in 0.28s
     Running `target/debug/display_manually`
TestDebug: 1 2
```


**example for Complex structure**
```Rust
use std::fmt;

struct Complex {
    x: f64,
    y: f64,
}

impl Complex {
    fn new(x: f64, y: f64) -> Self {
        Complex {
            x,
            y,
        }
    }
}

impl fmt::Display for Complex {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} + {}i", self.x, self.y)
    }
}

impl fmt::Debug for Complex {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Complex {{ real: {} imag: {} }}", self.x , self.y)
    }
}

fn main() {
    let t = Complex::new(3.3, 7.2);
    println!("{}\n{:?}", t, t);
}
```

*output*
```bash
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/complex_display`
3.3 + 7.2i
Complex { real: 3.3 imag: 7.2 }
```

Each `write!` generates a `fmt::Result`.Proper handing of this requires dealing with all the results.Rust provides `?` operater for exactly this purpose.

*display for vector*

```Rust
use std::fmt;

struct List(Vec<i32>);

impl fmt::Display for List {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // Extract the value using the type indexing
        let vec = &self.0;
        
        write!(f, "[")?;

        for (index, v) in vec.iter().enumerate() {
            if index != 0 {
                write!(f, ", ")?;
            }
            write!(f, "{}: {}", index, v)?;
        }
        write!(f, "]")
    }
}

fn main() {
    let v = List(vec![1, 2, 3]);
    println!("{}", v);
}
```

*output*
```bash
[0: 1, 1: 2, 2: 3]
```

## Formatting
Formatting is specified via a format string:

- `format!("{}", foo)` -> `"3735928559"`
- `format!("0x{:X}", foo)` -> `"0xDEADBEEF"`
- `format!("0o{:o}", foo)` -> `"0o33653337357"`

We can pad with zeros to a width of 6 with `:06`

*example code Color*

```Rust
use std::fmt;

struct Color {
    r: i32,
    g: i32,
    b: i32,
}

impl Color {
    fn new(r: i32, g: i32, b: i32) -> Self {
        Color { r, g, b}
    }
}

impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "RGB ({}, {}, {}) 0x{:06X}", self.r, self.g, self.b, self.r * (1 << 16) + self.g * (1 << 8) + self.b )
    }
}

fn main() {
    let t1 = Color::new(128, 255, 90);
    let t2 = Color::new(0, 3, 254);
    let t3 = Color::new(0, 0, 0);
    println!("{}\n{}\n{}", t1, t2, t3);
}
```






