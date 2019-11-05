---
title: Rust by example notes II
date: 2019-11-05 15:11
tags: 
    - Rust
    - Type
---

Rust by examples notes about primitives.

## Primitives
Rust provides access to a wide variety of `primitives`.

### scalar Types

- signed integers: `i8`, `i16`, `i32`, `i64` and `isize`(pointer size)
- unsigned integers: `u8`, `u16`, `u32`, `u64` and `usize`(pointer size)
- floating point: f32, f64
- `char` Unicode scalar values like `'a'`, `'α'` and `'∞'`(4 bytes each)
- `bool` either `true` and `false`
- and the unit type `()`, whose only possible value is an empty tuple: `()`

**note:** Despite the value of a unit type being a tuple, it is not considered a compound type because it does not contain multiple values.

### Compound Types
- arrys like [1, 2, 3]
- tuples like (1, true)

### Tuples
A tuple is a collection of values of different types. Tuples are constructed using parenthese `()`, and each tuple itself is a value with type signature `(T1, T2, ...)`, where `T1`, `T2` are the types of its members.Functions can use tuples to return multiple values, as tuples can hold any number of values.

```Rust
#[derive(Debug)]
struct Matrix(f32, f32, f32, f32);
let long_tuple = (1u8, 2u16, 3u32, 4u64,
                      -1i8, -2i16, -3i32, -4i64,
                      0.1f32, 0.2f64,
                      'a', true);
```

*Values can be extracted from the tuple using tuple indexing,such as`long_tuple.0`,`long_tuple.1`*

*Tuples can be tuple members*

```Rust
let tuple_of_tuples = ((1u8, 2u16, 2u32), (4u64, -1i8), -2i16);
```

*`let` can be used to bind the members of a tuple to variables.*

```Rust
let tuple = (1, true, 1.2);
let (a , b, c) = tuple;
```

*Display for tuples*

```Rust
use std::fmt::{Display, Formatter, Result};

struct Matrix(f32, f32, f32, f32);

fn transpose(x: &Matrix) -> Matrix {
    Matrix(x.0, x.2, x.1, x.3)
}

impl Display for Matrix {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        write!(f, "( {} {} )\n", &self.0, &self.1)?;
        write!(f, "( {} {} )", &self.2, &self.3)
    }
}


fn main() {
    let matrix = Matrix(1.1, 1.2, 2.1, 2.2);
    println!("Matrix:\n{}", matrix);
    println!("Transpose:\n{}", transpose(&matrix));
    println!("Matrix:\n{}", matrix);
}
```

*output*

```bash
Matrix:
( 1.1 1.2 )
( 2.1 2.2 )
Transpose:
( 1.1 2.1 )
( 1.2 2.2 )
Matrix:
( 1.1 1.2 )
( 2.1 2.2 )
```

### Arrays and Slices
An array is a collection of objects of the same type `T`, stored in contiguous memory. Arrays are created using brackets `[]`, and **their size, which is know at compile time**, is part of  their signature `[T; size]`.

Slices are similar to arrays, but **their size is not know at compile time**. Instead, a slice is a **two-word** object, the first word is a pointer to the data, the second word is the length of the slice. The word size is the same as usize, determined by archiecture eg 64 bits on x86-64.Slices can be used to borrow a section of an array, and have type signature `&[T]`.

*elements can be initalized to the same value*

```Rust
let x: [i32; 500] = [0; 500]
```

## Custom Types
Rust custom data types are formed mainly through the two keywords:

- `struct`: define a structure
- `enum`: define an enumeration

### Structures
There are three types of structures ("struct") that can be created using the `struct` keyword:

- Tuple structs, which are, basically, named tuples.

```Rust
struct Pair(i32, i32);
let pair = Pair(1, 2);
// Destructure a tuple struct 
let Pair(integer, decimal) = pair;
```
- The classic `C structs`.
````Rust
struct Point {
    x: f32,
    y: f32,
}
let point = Point { 1, 1.1};
// Destructure the point using a `let` binding
let Point { x: my_x, y: my_y } = point;
````
- Unit structs, which are field-less, are useful for generics.

```Rust
struct Nil;
let _nil = Nil;
```

**Note**: Structs can be reused as fields of another struct.

*rectangle struct example*

```Rust
struct Point {
    x: f64,
    y: f64,
}

#[allow(dead_code)]
struct Rectangle {
    p1: Point,
    p2: Point,
}

fn rect_area(rectangle: &Rectangle) -> f64 {
    let Rectangle {
        p1: Point {x: x1, y: y1}, 
        p2: Point {x: x2, y: y2}
    } = rectangle;
    //println!("{} {} {}", y2, y1, y2 - y1);
    ((x2 - x1) * (y2 - y1))
}

fn square(p: &Point, x: f64) -> Rectangle {
    Rectangle {
        p1: Point { x: p.x, y: p.y, },
        p2: Point { x: p.x + x, y: p.y + x, },
    }
}

fn main() {
    let point: Point = Point { x: 0.3, y: 0.4 };
    let x = 0.5;
    let rectangle = square(&point, x);
    println!("{}",rect_area(&rectangle));
}
```

*output*

```
0.5
```

## Enums
The `enum` keyword allows the creation of a type which may be one of a few different variants. Any variant which is valid as a `struct` is also valid as `enum`.

*example*

```Rust
// Create an `enum` to classify a web event. Note how both
// names and type information together specify the variant:
// `PageLoad != PageUnload` and `KeyPress(char) != Paste(String)`.
// Each is different and independent.
enum WebEvent {
    // An `enum` may either be `unit-like`,
    PageLoad,
    PageUnload,
    // like tuple structs,
    KeyPress(char),
    Paste(String),
    // or c-like structures.
    Click { x: i64, y: i64 },
}
```

### use
The `use` declaration can be used so manual scoping isn't needed:

```Rust
enum Status {
    A,
    B,
}
```

*no use*

```Rust
let status = Status::A;
```

*use*

```Rust
use crate::Status::{A, B};
let status = A;
```

### C-like
`enum` can also be used as C-like enums.

```Rust
// An attribute to hide warnings for unused code.
#![allow(dead_code)]

// enum with implicit discriminator (starts at 0)
enum Number {
    Zero,
    One,
    Two,
}

// enum with explicit discriminator
enum Color {
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff,
}

fn main() {
    // `enums` can be cast as integers.
    println!("zero is {}", Number::Zero as i32);
    println!("one is {}", Number::One as i32);

    println!("roses are #{:06x}", Color::Red as i32);
    println!("violets are #{:06x}", Color::Blue as i32);
}
```

### Linked-list
Cons: Tuple struct that wraps an element and a pointer to the next node

```Rust
use crate::List::*;

#[derive(Debug)]
enum List {
    Cons(u32, Box<List>),
    Nil,
}

impl List {
    fn new() -> List {
        Nil
    }

    fn prepend(self, elem: u32) -> List {
        Cons(elem, Box::new(self))
    }

    fn len(&self) -> u32 {
        match *self {
            Cons(_, ref tail) => 1 + tail.len(),
            Nil => 0,
        }
    }

    fn stringify(&self) -> String {
        match *self {
            Cons(head, ref tail) => {
                format!("{}, {}", head, tail.stringify())
            },
            Nil => {
                format!("Nil")
            },
        }
    }
}

fn main() {
    let mut list = List::new();

    list = list.prepend(1);
    list = list.prepend(2);
    list = list.prepend(3);
    
    println!("{}", list.len());
    println!("{}", list.stringify());
}
```

### constants
Rust has two different types of constants which can be declared in any scope including global. Both require explicit type annotation:

- `const`: An unchanged value (the common case).
- `static`: A possibly mutable variable with `'static` lifetime. The staic lifetime is inferred and does not have to be specified. Accessing or modifying a mutable static variable is `unsafe`.

## Type conversation
The generic conversations will use the `From` and `Into` traits.

### `From` and `Into`
`From` and `Into` traits are inherently linked. If your are able to convert type A from type B, then it should be easy to believe that we should be able to convert type B to type A.

The `From` trait allows for a type to define how to create itself from another type, hence providing a very simple mechanism for converting between several types.

```Rust
let my_str = "hi";
let my_string = String::from(my_str);
```

The `Into` trait is simply the reciprocal of the `From` trait. That is, if you have implemented the `From` trait for your type you get the `Into` implementation for free.

```Rust
use std::convert::From;

#[derive(Debug)]
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let num = Number::from(100);
    println!("{:?}", num);
    let int = 5;
    let num: Number = int.into();
    println!("{:?}", num);
}
```

### To and from Strings

#### Converting to String
To convert any type to `String` is as simple as implenting the `ToString` trait for the type. Rather than doing so directly, you should implement the `fmt::Display` trait which automagically provides `ToString` and also allows printing the types as discussed in the section on `print!`.

#### Parsing a String
```Rust
fn main() {
    let parsed: i32 = "5".parse().unwrap();
    let turbo_parsed = "10".parse::<i32>().unwrap();

    let sum = parsed + turbo_parsed;
    println!("Sum: {:?}", sum);
}
```






















   
