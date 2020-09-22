---
title: Returning Traits with `dyn`
date: 2020-9-22 21:21
tags: 
    - Rust
    - dyn
    - trait
---

Let's see the following codes:

```rust
use rand::prelude::*;

struct Sheep {}
struct Cow {}

trait Animal {
    fn noise(&self) -> &'static str;
}

impl Animal for Sheep {
    fn noise(&self) -> &'static str {
        "baaaaaah!"
    }
}

impl Animal for Cow {
    fn noise(&self) -> &'static str {
        "mooooooo!"
    }
}

fn random_animal(random_number: f64) -> Box<dyn Animal> {
    if random_number < 0.5 {
        Box::new(Sheep {})
    } else {
        Box::new(Cow {})
    }
}

fn choose_cow(random_number: f64) -> impl Animal {
    println!("choose_cow::The number is {}.", random_number);
    Cow {
    }
}

fn main() {
    let mut rng = rand::thread_rng();
    let random_number = rng.gen();
    let animal = random_animal(random_number);
    println!("You've randomly chosen an animal, and it says {}", animal.noise());

    let animal = choose_cow(random_number);
    println!("You've chosen a cow, and it says {}", animal.noise());
}
```

As the codes saying, if we're returning **a single type** for our functions, we can use `impl Trait`, just as:

```rust
fn choose_cow(random_number: f64) -> impl Animal {
    println!("choose_cow::The number is {}.", random_number);
    Cow {
    }
}
```

However, if function returns **multiable types**, it doesn't work. Because the Rust compiler needs to know how much space every functions's return type requiers. This means the return type of a function must have a statically known size. We can't write a function that returns `Animal`, because the different implementions will need different amounts of memory.

Instead of returning a trait object directly, our functions return a `Box` which contains some `Animal`.  A `box` is just a reference to some memory in the heap. Because a reference has a statically-known size, and the compiler can guarantee it points to a heap-allocated `Animal`, we can return a trait from our function!

Rust tries to be as explicit as possible whenever it allocates memory on the heap. So if your function returns a **pointer-to-trait-on-heap** in this way, you need to write the return type with the dyn keyword, e.g. `Box<dyn Animal>`.

