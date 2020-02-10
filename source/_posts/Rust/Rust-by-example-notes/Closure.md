---
title: Closure
date: 2019-11-27 18:44
tags: 
    - Rust
    - closure
---

## About closure
Closure in Rust, also called lamba expressions or lambdas, are functions that can capture the enclosing environment.

Some characterisitics of closures include:

- using `||` instead of `()` around input variables.
- optional body delimination `{}` for a single expression (mandatory otherwise).
- the ability to capture the outer environment variables

*example*

```Rust
fn main() {
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred = |i| i + 1;

    println!("{}", closure_annotated(1));
    println!("{}", closure_inferred(2));

    let x = || {
        println!("Hi xxxx");
    };
    x();
}
```

### Capturing
Capturing can flexibly adapt to the use case, sometimes moving and sometimes borrowing. Closures can capture variables:

- by reference: `&T`
- by mutable reference: `&mut T`
- by value: `T`

Closures preferentially capture variables by reference and only go lower when required.

*example*
```Rust
fn main() {
    use std::mem;

    // &T
    let color = "red";
    let print = || println!("color is {}", color);
    print();
    print();

    // &mut T
    let mut count = 0;
    let mut inc = || {
        count += 1;
        println!("count is {}", count);
    };
    inc();
    inc();
    println!("count: {}", count);

    // A non-copy type
    let movable = Box::new(3);
    let consume = || {
        println!("movable: {:?}", movable);
        mem::drop(movable);
    };
    consume();
    //consume();
}
```

Howerver, if we do this:

```Rust
fn main() {
    use std::mem;

    // A non-copy type
    let movable = Box::new(3);
    let consume = || {
        println!("movable: {:?}", movable);
        mem::drop(movable);
    };
    consume();
    consume();
}
```

We would get:

```bash
error[E0382]: use of moved value: `consume`
  --> src/main.rs:11:5
   |
10 |     consume();
   |     ------- value moved here
11 |     consume();
   |     ^^^^^^^ value used here after move
   |
note: closure cannot be invoked more than once because it moves the variable `movable` out of its environment
  --> src/main.rs:8:19
   |
8  |         mem::drop(movable);
   |                   ^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: Could not compile `capturing_test`.

To learn more, run the command again with --verbose.
```

`mem::drop` requires `T` so this mut take by value. A copy type would copy into the closure leaving the original untouched. A non-copy mut move and so `movable` immediately moves into the closure.

#### `move`
Using `move` before vertical pipes forces closures to take ownership of captured variables:

*example*
```Rust
fn main() {
    let vec = vec![1, 2, 3];

    let contains = move |needle| vec.contains(needle);

    println!("{}", contains(&1));
    println!("{}", contains(&2));

}
```

*output*
```
true
true
```

If we call  `vec.len()` later:
```Rust
fn main() {
    let vec = vec![1, 2, 3];

    let contains = move |needle| vec.contains(needle);

    println!("{}", contains(&1));
    println!("{}", contains(&2));

    println!("{}", vec.len());
}
```

We would get:
```bash
error[E0382]: borrow of moved value: `vec`
 --> src/main.rs:9:20
  |
2 |     let vec = vec![1, 2, 3];
  |         --- move occurs because `vec` has type `std::vec::Vec<i32>`, which does not implement the `Copy` trait
3 | 
4 |     let contains = move |needle| vec.contains(needle);
  |                    ------------- --- variable moved due to use in closure
  |                    |
  |                    value moved into closure here
...
9 |     println!("{}", vec.len());
  |                    ^^^ value borrowed here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: Could not compile `movable_closure_test`.
```

It's no doubt that `vec` has been moved into the closure.

### As input parameters
While Rust choose how to capture variables on the fly mostly without type annotaions, this ambiguity is not allowed when writing functions. When taking a closure as an input parameter, the closure's complete type must be annotated using one of a few `traits`. In order of decreasing restriction, they are:

- `Fn`: the closure captures by reference(`&T`)
- `FnMut`: the closure captures by mutable reference(`&mut T`)
- `FnOnce`: the closure captures by value(`T`)

For instance, consider a parameter annotated as `FnOnce`. This specifies that the closure may caputure by `&T`, `&mutT`, or `T`, but ultimately choose based on how the captured variables are used in the closure.

This is because if a move is possible, then any type of borrow should also be possible. Note that reverse is not true. If the paramenter is annotated as `Fn`, then capturing varibales by `&mut T` or `T` are not allowed.

*example*
```Rust
fn apply<F>(f: F) where
    F: FnOnce() {

    f();
}

fn apply_to_3<F>(f: F) -> i32 where
    F: Fn(i32) -> i32 {

    f(3)
}

fn main() {
    use std::mem;

    let greeting = "hello";
    let mut farewell = "goodbye".to_owned();

    let dairy = || {
        // greeting is by reference: requries Fn
        println!("I said {}", greeting);

        // Mutation forces `farewell` to be captured by mutable reference 
        // Now require FnMut
        farewell.push_str("!!!");
        println!("Then I screamed {}", farewell);
        
        // Now require FnOnce
        mem::drop(farewell);
    };

    apply(dairy);

    let double = |x| x * 2;
    println!("{}", apply_to_3(double));   
}

```


### Type anonymity
Using a closure as a parameter requires **generics**.This is necessary because of how thet are defined:

```Rust
// `F` must be generic.
fn apply<F>(f: F) where
    F: FnOnce() {
    f();
}
```

When a closure is defined, the compiler implicitly creates a new anonymous structure to store the captured variables inside, meanwhile implementing the functionality via one of the `traits`: `Fn`, `FnMut` or `FnOnce` for this unknown type. This type is assigned to the variable which is stored until calling.

Since this new type if of unknown type, any usage in a function will require generics. However, an unbounded type parameter <T> would still be ambigious and not be allowed. Thus, bounding by one of the `traits`: `Fn`, `FnMut` or `FnOnce`(which it implements) is sufficient to specify its type.

### Input functions
If you declare a function that takes a closure as a parameter, then any *function* that satisfies the trait bound of that closure can be passed as parameter.

*example*
```Rust
fn call_me<F>(f: F) 
where 
    F: Fn()
{
    f();
}

fn function() {
    println!("Hi, I'm a function.");
}

fn main() {
    let closure = | | println!("Hi, I'm a closure.");
    call_me(closure);
    call_me(function);
}
```

`function` could also be used as a parameter as the same as `closure`.

### As output parameters
Returning closures as output parameters are also possible. However, anoymous closure types are, by definition, unknown, so we have to use `impl Trait` to return them.

*example*
```rust
fn create_fn() -> impl Fn() {
    let text = "Fn".to_owned();
    move || println!("{}", text)
}

fn create_fnmut() -> impl FnMut() {
    let text = "FnMut".to_owned();
    move || println!("{}", text)
}

fn main() {
    let fn_plain = create_fn();
    let mut fn_mut = create_fnmut();
    fn_plain();
    fn_mut();
}
```



























