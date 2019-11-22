---
title: Substrate-workshop notes
date: 2019-11-22 12:13
tags: 
    - Rust
    - Substrate
    - blcokchain
---

## Runtime 
The Runtime is block execution logic of a blockchain, sometimes referred to as the state transition function `STF`. In *Substrate*, this is stored on-chain in an implementation-neutal(语言无关的), machine-executable(机器可执行的) format as a WebAssembly binary.

**Other system**

- Ethereum(human-readable format)
- Bitcoin(not at all)

 The runtime is composed of multiple features and functionalities which work together to power your blockchain. Things like:

 -   Account Management
 -   Token Balances
 -   Governance
 -   Runtime Upgrades
 -   and more...
 
## Creating a Module
First, we need to create a module for our runtime. For that we will work with an empty module template which we will place in a new `substratekitties.rs` file:

```
substratekitties
|
+-- runtime
    |
    +-- src
        |
        +-- lib.rs
        |
        +-- * substratekitties.rs
    |
    +-- template.rs 
```

*substratekitties.rs*

```rust
use support::{decl_storage, decl_module};

pub trait Trait: system::Trait{
    
}

decl_storage! {
    trait Store for Module<T: Trait> as KittyStorage {
        // Declare storge and getter functions here
    }
}

decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Declare public functions here
    }
}
```

This template allows us to start writing the most basic parts of our module, the public functions and the storage.

*Note: The line `trait Store for Module<T: Trait> as NAME` is macro magic. That line as written is not valid Rust, but it gets converted to valid Rust code through the `decl_storage! macro`.*

## Updating our Runtime
For each module, we should:

- Import the Rust file containing the module
- Implement its trait
- Include the module into the `construct_runtime!` macro

**Firstly**, import the `substratekitties.rs`.
We should add this line into the `lib.rs`.

```rust
// lib.rs

/// Index of a block number in the chain.
pub type BlockNumber = u64;

/// Index of an account's extrinsic in the chain.
pub type Nonce = u64;

// Add this line
mod substratekitties;
```

*Secondlly*, implement the traits. Our `Triat` implementation is very simple, because we haven't defined anything in it yet.

```rust
// lib.rs

impl sudo::Trait for Runtime {
	/// The uniquitous event type.
	type Event = Event;
	type Proposal = Call;
}

// Add the implementation here
impl substratekitties::Trait for Runtime {
}
```

**Finally**, add this line at the end of our `construct_runtime!` definition:

```
// lib.rs

construct_runtime!(
	pub enum Runtime with Log(InternalLog: DigestItem<Hash, AuthorityId, AuthoritySignature>) where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		System: system::{default, Log(ChangesTrieRoot)},
		Timestamp: timestamp::{Module, Call, Storage, Config<T>, Inherent},
		Consensus: consensus::{Module, Call, Storage, Config<T>, Log(AuthoritiesChange), Inherent},
		Aura: aura::{Module},
		Indices: indices,
		Balances: balances,
		Sudo: sudo,
		
		// Add this line
		Substratekitties: substratekitties::{Module, Call, Storage},
	}
);
```

Note than we have added three `types` to this definition(`Module`, `Call`, `Storage`), all of which are produced by the macros defined in our template.

## Creating a Storage Value
Let's add a function which stores a variable. Substrate natively supports all the primitive types avaliable in Rust(`bool`, `u8`, `u32`, etc..) and custom types sepcific to Substrate(`AccountId`m `BlockNumber`, `Hash`, [etc..](https://polkadot.js.org/api/types/#codec-types) )

We can declare a simple storage item like this:

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        MyU32: u32;
        MyBool get(my_bool_getter): bool;
    }
}
```

Here we have defined two variables: a `u32` and a `bool` with a getter function named `my_bool_getter`. The `get` parameter is **optional**, but i*f you add it to your storage item it will expose a getter function with the name specified(`fn getter_name() -> Type`)*.

To store these basic values, we need to import the `support::StorageValue` module.

The function used to access a `StorageValue` are defined in [StorageValue](https://substrate.dev/rustdocs/v1.0/srml_support/storage/trait.StorageValue.html).

Now, create a storage value called `Value` which stores as `u64`.

```rust
decl_storage! {
    trait Store for Module<T: Trait> as KittyStorage {
        // Declare storage and getter functions here
        Value: u64;
    }
}
```

## Storing a Value
Now that we have our *storage value* declared in our runtime, we can actually create a *function* to push a value to it.

### Declaring a public function
We need to define runtime functions that will set and modify our storage values. This can be done within our `decl_module!` macro, which declares all the entry points that your module handles.

```rust
// Add these imports: 
//
use support::{dispatch::Result, StorageValue};
use system::ensure_signed;

decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Declare public functions here
        fn set_value(origin, value: u64) -> Result {
            let _sender = ensure_signed(origin)?;

            <Value<T>>::put(value);

            Ok(())
        } 
    }
}
```

### Function Structure
Module functions exposed here should always take the form:

```rust
fn foo(origin, bar: Bar, baz: Baz, ...) -> Result;
```

#### Origin
The first argument of these function is always `origin`. `origin` contains information about where the call originated from. This is generally split into three groups:

- Public calls that are signed by an external account.
- Root calls that are allowed to be made only by the governance system.
- Inherent calls that are allowed to be made only by the block authors and validators.

#### Checking for a Signed Message
The first argument in any of these module functions is the `origin`. There are three convenience call in `system` that do the matching for your and return a convenient result: `ensure_signed`, `ensure_root` and `ensire_inherent`. **You should always match against them as the first thing you do in your function.**

We can use the `ensure_signed()` function from `system::ensure_signed` to check the origin, and "ensure" that the messaged is signed by a valid account.

## Storage Mapping
Our last runtime only allowed us to **store a single value across all users** of our blockchain. As we start thinking toward our collectables chain, it makes sense to add support to have their own value stored.

To enable this, we will replace our single value storage with a storage mapping.

The functions used to access a StorageMap are in [StorageMap](https://substrate.dev/rustdocs/v1.0/srml_support/storage/trait.StorageMap.html) 

Now our storage example is updated to store a map from `AccountId` to a `u64`.

```rust
// change `StorageValue` to `StorageMap`

decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Declare public functions here
        fn set_value(origin, value: u64) -> Result {
            let sender = ensure_signed(origin)?;

            <Value<T>>::insert(sender, value);

            Ok(())
        } 
    }
}
```

## Storing a Structure
We can define a `struct` for digital kitties and store them in our runtime storage like so:

```rust
#[derive(Encode, Decode, Default, Clone, PartialEq)]
#[cfg_attr(feature = "std", derive(Debug))]
pub struct Kitty<Hash, Balance> {
    id: Hash,
    dna: Hash,
    price: Balance,
    gen: u64,
}
```

Note: To use the custom `Encode` and `Decode` traits, you will need to import them from the `parity_codec` crate:

```rust
use parity_codec::{Encode, Decode};
```

We define our example struct using a generic as one of the types that we store. This will be important when trying to use custom Substrate types like `AccountId` or `Balance` within our struct as we wil need to pass in these types every time we use our struct.

So, if we wanted to store a `Balance` in `some_generic` and `Hash` in `some_other_generic`, we wiuld need to define our storage item like this:

```rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        MyItem: map T::AccountId => MyStruct<T::Balance, T::Hash>;
    }
}
```

For the purpose of clarity, we will name a generic type for `T::AccountId` as `AccountId` and `T::Balance` as `Balance`.

[Read More](https://github.com/paritytech/substrate/wiki/FAQ) 

For our example:

```rust
decl_storage! {
    trait Store for Module<T: Trait> as KittyStorage {
        // Declare storage and getter functions here
        OwnedKitty get(kitty_of_owner): map T::AccountId => Kitty<T::Hash, T::Balance>;
    }
}
```

We update the storage item to sotre a `Kitty<T::Hash, T::Balance>`, add a getter function named `kitty_of_owner`.

Now, we have initialized our custom struct in our runtime storage, we can now push values and modify it.

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Declare public functions here
        fn create_kitty(origin) -> Result {
            let sender = ensure_signed(origin)?;

            let new_kitty = Kitty {
                id: <T as system::Trait>::Hashing::hash_of(&0),
                dna: <T as system::Trait>::Hashing::hash_of(&0),
                price: <T::Balance as As<u64>>::sa(0),
                gen: 0,
            };

            <OwnedKitty<T>>::insert(sender, new_kitty);
            
            Ok(())
        }
        
    }
}
```

## Generating Random Data
Now, we allowed each user to create their own kitty. However, they weren't very unique. Let's fix that.

### Generating a Random Seed
In order to tell these kitties apart, we need to generate a unique `id` for each kitty and some random `dna`.

We can securely fetch some randomness from our chain using the `system` module:

```rust
<system::Module<T>>::random_seed()
```

Substrate uses a safe mixing algorithm that uses the entropy of previous blocks to generate new random data for each subsequent block.

However, since it is dependent on previous blocks, it can take over 80 blocks to fully warm up, and you may notice the seed will not change until then.

### Using a Nonce
Since the random seed does not change for multiple transactions in the same block, and since it may not even generate a random seed for the first 80 blocks, it is important that we also introduce a `nonce` which our module can manage. Furthermore, we can also user a user specific property like the `AccountId` to introduce a bit more entropy.

### Hashing Data
A random number generator:

```rust
let sender = ensure_signed(origin)?;
            let nonce = <Nonce<T>>::get();
            let random_seed = <system::Module<T>>::random_seed();

            let random_hash = (random_seed, &sender, nonce).using_encoded(<T as system::Trait>::Hashing::hash);
```

We can use this `random_hash` to populate both the `id` and `dna` for our kitty.

`using_encoded`: Convert self to a slice and then *invoke the given closure with it*.

### Checking for Collision
The `id` on the `Kitty` should be unique. We can do this with a new storage item `Kitties` which will be a mapping from `id`(`Hash`) to the `Kitty` object.

```rust
Kitties: map T::Hash => Kitty<T::Hash, T::Balance>;
```

For this object, we can easily check for collisions by simply checking whether this storage item already contains a mapping using a particular `id`.

```rust
ensure!(!<Kitties<T>>::exists(random_hash), "This id is already exists");
```

### Updating the code
So we should update our storage module. First, we should add tewo new kitty storage item.

- `Kitties` 
	point from our kitty's id to the `Kitty` object
- `KittyOwner` 
	point from our kitty's id to the owner
	
Then update the `OwnedKitty` storage below to store the kitty's id rather than the `Kitty` object.

Finally, add a `u64` value named `Nonce`.

```rust
decl_storage! {
    trait Store for Module<T: Trait> as KittyStorage {
        // Declare storage and getter functions here
        Kitties: map T::Hash => Kitty<T::Hash, T::Balance>;
        KittyOwner: map T::Hash => Option<T::AccountId>;
        
        OwnedKitty get(kitty_of_owner): map T::AccountId => T::Hash;

        Nonce: u64;
    }
}
```

The `create_kitty` should be updated too:

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Declare public functions here
        fn create_kitty(origin) -> Result {
            let sender = ensure_signed(origin)?;
            let nonce = <Nonce<T>>::get();
            let random_seed = <system::Module<T>>::random_seed();

            let random_hash = (random_seed, &sender, nonce).using_encoded(<T as system::Trait>::Hashing::hash);

            ensure!(!<Kitties<T>>::exists(random_hash), "This id is already exists");

            <Nonce<T>>::mutate(|n| *n += 1);
            
            let new_kitty = Kitty {
                id: random_hash,
                dna: random_hash,
                price: <T::Balance as As<u64>>::sa(0),
                gen: 0,
            };

            <Kitties<T>>::insert(random_hash, new_kitty);
            <KittyOwner<T>>::insert(random_hash, &sender);

            <OwnedKitty<T>>::insert(&sender, random_hash);
            
            
            Ok(())
        }
        
    }
}
```

`Nonce` will be a new item in our storage which we will *simply increment whenever we use it*.

## Creating an Event
On Substrate, **even though a transaction may be finalized, it does not necessarily imply that the function executed by that transaction fully succeed.**

To know that, we should **emit an `Event` at the end of the function** to not only report success, but to tell the "off-chain world" that some particular state transition has happened.

### Declaring an Event
`decl_event!` macro, example of an event declaration:

```rust
decl_event!(
    pub enum Event<T>
    where
        <T as system::Trait>::AccountId,
        <T as system::Trait>::Balance
    {
        MyEvent(u32, Balance),
        MyOtherEvent(Balance, AccountId),
    }
);
```

In our kitty-example:

```rust
decl_event! {
    pub enum Event<T>
    where
        <T as system::Trait>::AccountId,
        <T as system::Trait>::Hash,
    {
        Created(AccountId, Hash),
    }
}
```

If we want to use some custom Substrate types, we need to integerate generics into our event definition.

### Adding an Event
The decl_event! macro will generate a new Event type which you will need to expose in your module. This type will need to inherit some traits like so:

```rust
pub trait Trait: balances::Trait {
    type Event: From<Event<Self>> + Into<<Self as system::Trait>::Event>;
}
```

### Depositing an Event
In order to use events within your runtime, you need to add a function which deposits those events. The `decl_module!` macro can automatically add a default implementation of this to your module.

Add this to the `decl_module`:

```rust
fn deposit_event<T>() = default;
```
If you do not use any generics:

```rust
fn deposit_event() = default;
```

### Calling `deposit_event()`
Just provide the values that go along with our Event `definition` at the end of our function.

```rust
let my_value = 1337;
let my_balance = <T::Balance as As<u64>>::sa(1337);

Self::deposit_event(RawEvent::MyEvent(my_value, my_balance));
```

So to our projects:

```rust
 Self::deposit_event(RawEvent::Created(sender, random_hash));
```


### Updating `lib.rs` to Include Events
In the module `Trait` implementation:

```rust
// `lib.rs`

impl mymodule::Trait for Runtime {
    type Event = Event;
}
```

Include the `Event` or `Event<T>` type to the module's definition in the `construct_runtime!` macro.

```rust
construct_runtime!(
    pub enum Runtime with Log(InternalLog: DigestItem<Hash, Ed25519AuthorityId>) where
        Block = Block,
        NodeBlock = opaque::Block,
        InherentData = BasicInherentData
    {
        ...
        MyModule: mymodule::{Module, Call, Storage, Event<T>},
    }
);
```

#### Why we need events?
Followings are some of my understandings:

Substrate runtime module does not support `println!` macros for us to check the output from Substrate. However, we can deposit events from our substrate code and see it in polkadot.js apps. In **extrinsics** menu, we make a extrinsic which we built from our runtime and see events in the **explorer** menu.

## Tracking All Kitties

### Verify First, Write Last
There's big difference between Substrate and Etherenum. On Ethereum, if at any point your transaction fails (error, out of gas, etc...), the state of your smart contract will be unaffected. Howerver, on Substrate this is not the case. *As soon as a transaction starts to modify the storage of the blockchain, those changes are **parmanent**, even if the transaction would fail at a later time during runtime execution.*

As a Substrate runtime developer, we must follow "Verify first, write last" pattern.

### Creating a List
Substrate does support lists in the form of an [EnumerableStorageMap](https://substrate.dev/rustdocs/v1.0/srml_support/storage/trait.EnumerableStorageMap.html).

In runtime development, list iteration is, generally speaking, dangerous. Unless explicitly guarded against, **runtime functions which enumerate a list will add O(N) complexity, but only charge O(1) fees**. As a result, the chain can be vulnerable to attacks. Furthermore, if the lists you iterate over are large or even unbounded, **your runtime may need more time to process the list than what is allocated between blocks. This means that a block producer may not even be able to create new blocks!**

For this reason, we will not use any list iteration in our runtime logic. Instead, we will emulate an enumerable map with a mapping and a counter like so:

```rust
decl_storage! {
    trait Store for Module<T: Trait> as KittyStorage {
        ...

        AllKittiesArray get(kitty_by_index): map u64 => T::Hash;
        AllKittiesCount get(all_kitties_count): u64;
        
        ...
    }
}
```

Here we are storing a list of kitty in our runtime represented by `T::Hash`.

(有了这两个item以后，我们可以通过all_kitties_count来获得当前的kitty数，然后根据index(`最后一个索引index = count - 1`)去索引对应的kitty，就可以追踪到所有的kitty了，即遍历`0 ~ count -1`的`index`就可以遍历所有的kitty了)。

### Checking for Overflow/Underflow
Overflow and underflows are an easy way to cause our runtime to panic or for our storage to get messed up. We must always be proactive about checking for possible runtime errors before we make changes to our state. Ulike Ethereum, when a transaction fails, the state is **NOT** reverted back to before the transaction, so it is your responsibility to **ensure that there are no side effect on error**.

Fortuanately, checking for these kinds of errors are quite simple in Rust where primitive number types have `checked_add()` and `checked_sub()` functions.

```rust
let all_kitties_count = Self::all_kitties_count();
let new_all_kitties_count = all_kitties_count.checked_add(1).ok_or("Overflow adding a new kitty")?;
```

Using `ok_or` is the same as writing:

```rust
let new_all_kitties_count = match all_kitties_count.check_add(1) {
    Some(x) => x,
    Err("Overflow adding a new kitty"),
};
```

Make sure to remember the `?` at the end.

### Updating our List in Storage
Now that we have checked that we can safely increament our list, we can finally push changes to our storage. Remember that when you update your list, the *"last index" of your list is one less than the count*. For example, in a list with 2 items, the first item is index 0, and the second item is index 1.

```rust
fn create_kitty(origin) -> Result {
    let sender = ensure_signed(origin)?;

    let all_kitties_count = Self::all_kitties_count();
    let new_all_kitties_count = all_kitties_count.checked_add(1).ok_or("Overflow addina new kitty")?
    
    ...

    // (`index` is `count - 1` = `new_all_kitties_count 1` = `all_kitties_count`)
    <AllKittiesArray<T>>::insert(all_kitties_count, random_hash); 
    <AllKittiesCount<T>>::put(new_all_kitties_count);
    <AllKittiesIndex<T>>::insert(random_hash, all_kitties_count)
    
    ...

    Ok(())
}    
```

First, we get the current `AllKittiesCount` value and store it in `all_kitties_count`. Then create a `new_all_kitties_count` by doing a `checked_add()` to increment `all_kitties_count`. We also map the index(`all_kitties_count` = `new_kitties_count -1`, remember the `index` is `count -1` ) to the kitty(`random_hash`).

### Deleting From Our List
One problem that this `map` and `count` pattern introduces is holes in our list when we try to remove elements from the middle. Fortunately, the order of the list we want to manage in our example is not important, so we can use a "swap and pop" method to efficiently mitigate this issue.

The "swap and pop" method switches the position of the item we want to remove and the last item in our list. Then, we can simply remove the last item without introducing any holes to our list.

Rather than run a loop to find the index of the item we want to remove each time we remove an item, we will use a little extra storage to keep track of each item and its position in our list.

```rust
AllKittiesIndex: map T::Hash => u64;
```

（简单来讲其实就是每次先交换要删除的kitty和整个list里最后一个kitty，交换完后把最后一个kitty删掉就好，这样一来就不会因为删除在list里留下空缺。同时，我们用`AllKittiesIndex`这个数据结构来映射kitty和index，这样就不需要每次删除kitty的时候还要遍历整个list去找它的index）

## Owning Multiple Kitties
Right now our storage can only track one kitty per user, howerver one user can own multiple kitties.

>Note: 其实这种说法并不准确，虽然对于每个user只能看到最新的一只kitty，但是实际上通过某只kitty还是可以追踪到它的owner的。 (Though every user can only check the last kitty he has, we can find the kitty's owner by `KittyOwner`.)

### Using tuples to emulate higher order arrays
We could use a tuple to represent ownership of multiple items across multiple users.

Here is how we could build a "kitty list" unique to each person using such a structure:

```rust
OwnedKittiesArray get(kitty_of_owner_by_index): map (T::AccountId, u64) => T::Hash;
OwnedKittiesCount get(owned_kitty_count): map T::AccountId => u64;
```

This should emulate a more standard two-dimensional array like:

```
OwnedKittiesArray[AccountId][index] -> (one)kitty
```
        
Also we can get the number of kitties for a user like:        

```rust
OwnedKittiesArray[AccountId].length() = owned_kitty_count() = OwnedKittiesCount[AccountId]
```

### Relative Index
Just as before, we can optimize the computational work our runtime needs to do by indexing the location of items. The general approach to this would be to reserve the mapping of `OwnedKittiesArray`:

```rust
// (T::AccountId, T::Hash) -> `index` in `OwnedKittiesArray`
OwnedKittiesIndex: map (T::AccountId, T::Hash) => u64;
```

Howerver, our kitties all have unique identifiers as a `Hash`, and cannot be owned by more than one user, we can actually simplify this structure:

```rust
OwnedKittiesIndex: map T::Hash => u64;
```

This index tells us for a given kitty, where to look in the *owners* array for that item.

## Refactoring our code
Within our runtime, we are able to include an implementation of our runtime module like so:

```rust
impl<T: Trait> Module<T> {
    // our function here    
}
```

Functions in this block are usually public interfaces or private functions. Public interfaces should be labeled `pub` and generally fall into inspector functions that do not write to storage and operation functions that do. Private functions are your usual private utilities unavailable to other modules.

You can call functions defined here using the `Self::function_name()` pattern you have seen before. Here is an intentionally overcomplicated example:

```rust
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        fn adder_to_storage(origin, num1: u32, num2: u32) -> Result {
            let _sender = ensure_signed(origin)?;
            let result = Self::_adder(num1, num2);

            Self::_store_value(result)?;

            Ok(())
        }
    }
}

impl<T: Trait> Module<T> {
    fn _adder(num1: u32, num2: u32) -> u32 {
        let final_answer = num1.checked_add(num2).ok_or("Overflow when adding")?;
    }

    fn _store_value(value: u32) -> Result {
        <myStorage<T>>::put(value);

        Ok(())
    }
}
```

Remember that we still need to follow a "verify first, write last" pattern, so it is important to not daisy chain private functions which do writes to storage where there is a chance one will throw an error.

So, in our example, we moved most of the logic to the function `mint`:

```rust
impl<T: Trait> Module<T> {
    fn mint(to: T::AccountId, kitty_id: T::Hash, new_kitty: Kitty<T::Hash, T::Balance>) -> Result {
        let owned_kitty_count = Self::owned_kitty_count(&to);
        let new_owned_kitty_count = owned_kitty_count.checked_add(1).ok_or("Overflow adding a new kitty to account Balance")?;
            
        let all_kitties_count = Self::all_kitties_count();
        let new_all_kitties_count = all_kitties_count.checked_add(1).ok_or("Overflow adding a new kitty")?;

        <Kitties<T>>::insert(kitty_id, new_kitty);
        <KittyOwner<T>>::insert(kitty_id, &to);

            // (`index` is `count - 1` = `new_all_kitties_count - 1` = `all_kitties_count`,)
        <AllKittiesArray<T>>::insert(all_kitties_count, kitty_id); 
        <AllKittiesCount<T>>::put(new_all_kitties_count);
        <AllKittiesIndex<T>>::insert(kitty_id, all_kitties_count);
        <OwnedKittiesArray<T>>::insert((to.clone(), owned_kitty_count), kitty_id);
        <OwnedKittiesCount<T>>::insert(&to, new_owned_kitty_count);
        <OwnedKittiesIndex<T>>::insert(kitty_id, owned_kitty_count);
        <OwnedKitty<T>>::insert(&to, kitty_id);
    
        Self::deposit_event(RawEvent::Created(to, kitty_id));
        
        Ok(())
    }
}
```

## Set the price of a Kitty
Now, every kitty has a `price` attribute that we have set it to `0` as defalut. If we want to set the price of a kitty, we will need to pull down the `Kitty` object, update the price, and push it back into the storage.


### Sanity Checks
Before doing this, we need to do sanity checks. Since we are going to start letting users call public functions that our runtime exposes, and that means opportunity for our users to give poor input or even maliciously. So if we are creating a function which updates the value of an object, the first thing we better do is make sure the object exists at all.

```rust
ensure!(<Kitties<T>>::exists(kitty_id), "This kitty does not exists.");
```

### Permissioned Functions
Although everyone could call our `create_kitty()` function with a message, only the owner of the kitty is allowed to set the price. For modifying a `Kitty`, we need to get the owner of the kitty, and ensure that it is the same as the `sender`.

KittyOwner stores a mapping to an `Option<T::AccountId>` since a given `Hash` may not point to a generated and owned Kitty yet. This means, whenever we fetch the owner of a kitty, we need to resolve the possibility that it returns None. This could be caused by bad user input or even some sort of problem with our runtime, but checking will help prevent these kinds of problems.

> 其实这里说的不是很准确，每个kitty应该都有owner，如果是输入错误那么实际上在第一次检查的时候，就已经发现这个kitty不存在了。(In fact, every kitty should have its owner, if it does not have a owner, it should not exist.)

```rust
// kitty's owner
let owner = Self::owner_of(kitty_id).ok_or("No owner for this kitty")?;
ensure!(owner == sender, "You are not the owner of the kitty");
```

So the `set_price` function looks like:

```rust
fn set_price(origin, kitty_id: T::Hash, new_price: T::Balance) -> Result {
    let sender = ensure_signed(origin)?;
    ensure!(<Kitties<T>>::exists(kitty_id), "This kitty does not exists.")

         // kitty's owner
    let owner = Self::owner_of(kitty_id).ok_or("No owner for this kitty")?;
    ensure!(owner == sender, "You are not the owner of the kitty")

    let mut kitty = Self::kitty(kitty_id);
    kitty.price = new_price

    <Kitties<T>>::insert(kitty_id, kitty)

    Self::deposit_event(RawEvent::PriceSet(sender, kitty_id, new_price));
         
    Ok(())
}
```

## Transferring a Kitty
Ownership is entirely managed by our storage, so a `transfer_kitty` function is really only modifying our existing storage to reflect the state. Here are the storage items we need to update:

- Change the global kitty owner
- Change the owned kitty count of each user
- Change the owned kitty index of the kitty
- Change the owned kitty map for each user

```rust
fn transfer(origin, to: T::AccountId, kitty_id: T::Hash) -> Result {
    let sender = ensure_signed(origin)?;

    let owner = Self::owner_of(kitty_id).ok_or("No owner for this kitty")?;
    ensure!(owner == sender, "You are not the owner of this kitty");

    Self::transfer_from(sender, to, kitty_id)?;
            
    Ok(())
}
```

```rust
fn buy_kitty(origin, kitty_id: T::Hash, max_price: T::Balance) -> Result {
    let sender = ensure_signed(origin)?;

    ensure!(<Kitties<T>>::exists(kitty_id), "This kitty does not exists.");

    let owner = Self::owner_of(kitty_id).ok_or("No owner for this kitty")?;

    let mut kitty = Self::kitty(kitty_id);
            
    let kitty_price = kitty.price;
    ensure!(!kitty_price.is_zero(), "Price is zero.");
    ensure!(kitty_price <= max_price, "The cat you want to by cost more than your max price");
            
    <balances::Module<T> as Currency<_>>::transfer(&sender, &owner, kitty_price)?;
    Self::transfer_from(owner.clone(), sender.clone(), kitty_id)
        .expect("`owner` is shown to own the kitty; \
                `owner` must have greater than 0 kitties, so transfer cannot cause underflow; \
                `all_kitty_count` shares the same type as `owned_kitty_count` \
                and minting ensure there won't ever be more than `max()` kitties, \
                which means transfer cannot cause an overflow; \
                qed");

    kitty.price = <T::Balance as As<u64>>::sa(0);

    Self::deposit_event(RawEvent::Bought(sender, owner, kitty_id, kitty_price));
            
    Ok(())    
}
```

## Buying a Kitty
First, make sure that the kitty is indeed for sale. To simplified our problem, just define that any kitty with default price of 0 is not for sale.

Then, we need to make a payment. So far our chain has been completely independent of our internal currency provided by the `Balances` module. The `Balances` module gives us access to completely manage the internal currency of every user, which means we need to be careful how we use it.

Fortunately, the `Balances` module expose a trait called `Currency` which implements a function called `transfer()` which allows you to safely transfer units from one account to another, checking for enough balance, overflow, underflow, and even account creation as a result of getting tokens.

```rust
<balances::Module<T> as Currency<_>>::transfer(&sender, &owner, kitty_price)?;
```
## Breeding a Kitty
Probably the most unique part of the origina; CryptoKitties game is the ability to breed new kitties from existing ones.

We have prepared our `Kitty` object with this in mind, introducing `dna` and `gen` which will be used in forming brand new kitty offspring.

In our runtime, DNA is a 256 bit hash, which is represented by as a bytearray in our code, and a hexadecimal string in our upcoming UI.

This means that there are 32 elements, each of which can be a value from 0 - 255. We will use these elements to determine which traits our kitties have. For example, the first index of the byte array can determine the color of our kitty(from a range of 256 colors); the nex element could represent the eye shape, etc...


```rust
fn breed_kitty(origin, kitty_id_1: T::Hash, kitty_id_2: T::Hash) -> Result {
    let sender = ensure_signed(origin)?
    ensure!(<Kitties<T>>::exists(kitty_id_1), "Kitty1 doenot exist");
    ensure!(<Kitties<T>>::exists(kitty_id_2), "Kitty2 doenot exist")
    let nonce = <Nonce<T>>::get();
    let random_hash = (<system::Module<T>>::random_seed(),sender, nonce).using_encoded(<T asystem::Trait>::Hashing::hash);
    
    let kitty_1 = Self::kitty(kitty_id_1);
    let kitty_2 = Self::kitty(kitty_id_2)
    let mut final_dna = kitty_1.dna;
    // 'Zips up' two iterators into a single iterator opairs. tuple
    for (i, (dna_2_element, r)) in kitty_2.dna.as_ref().it().zip(random_hash.as_ref().iter()).enumerate() {
        if r % 2 == 0 {
            final_dna.as_mut()[i] = *dna_2_element;
        }    
    
    let new_kitty = Kitty {
        id: random_hash,
        dna: final_dna,
        price: <T::Balance as As<u64>>::sa(0),
        gen: cmp::max(kitty_1.gen, kitty_2.gen) + 1,
    }
    Self::mint(sender, random_hash, new_kitty)?
    <Nonce<T>>::mutate(|n| *n += 1);
    
    Ok(())
}
```
