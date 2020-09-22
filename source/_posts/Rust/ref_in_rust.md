---
title: ref in Rust
date: 2020-9-22 21:21
tags: 
    - Rust
    - ref
---

When doing pattern matching or destructuring via `let` binding, the `ref` keyword can be used to take references to the field of a struct/tuple.

A `ref` borrow on the left side of an assignment is equivalent to an `&` borrow on the right side.

```rust
fn main() {
    let i = 3;
    {
        let borrow1 = &i;

        println!("borrow1: {}", borrow1);
    }
    
    {
        let ref borrow2 = i;
        println!("borrow2: {}", borrow2);
    }
}
```

Destructuring a struct:

```rust
struct Point {
    x: i32,
    y: i32
}

fn main() {
    let point = Point{x: 1, y: 2};

    let Point{x: xx, y: yy} = point;
    println!("{} {}", xx, yy);

    let Point{x: ref xx, y: ref yy} = point;
    println!("{} {}", *xx, *yy);
}
```
