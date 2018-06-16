---
layout: post
title:  "Dynamic Casting for Traits"
categories: rust
---

In Rust, traits are a powerful tool to use polymorphism, both static and dynamic. I'm going to skip the basics about the traits and just link to another blog post with a good explanation about static and dynamic dispatch in Rust: [Traits and Trait Objects in Rust](https://joshleeb.com/posts/rust-traits-and-trait-objects/).

Instead, I would like to do an experiment of making dynamic dispatch even more dynamic! Like in Java[^1].

## How Dynamic is a Dynamic Dispatch?

The way Rust dynamic dispatch works is via a special reference type, a trait object which provides a way to dispatch a call based on a concrete type. Again, I am going to skip the details, but the idea is that a trait object is represented internally by two pointers. The first pointer points to the data itself. The second pointer points to a virtual table where each "slot" is the address of the function.

Every time Rust compiler generates code to invoke a function on a trait object, the generated code would destructure the trait object into two pointers, the data pointer and virtual table pointer. Then, it would lookup the virtual table to get the address of the function to call (each function is assigned a unique slot in the table, so this is a simple indexing operation). Finally, the code would call that function, providing a pointer to the data as the first parameter (which becomes `&self` in the function).

However, what if we want to dispatch a call based on both a type of the interface we want to dispatch on and the concrete type? What does it mean and why would we even want that?

### Is it a Failure?

One example where this could be useful (which looks reasonable to me, but could be a totally terrible idea!) would be a [`failure`](https://crates.io/crates/failure) crate. This crate provides an error handling abstraction which, among other things, allows to build chains of errors and iterate these chains.

This is convenient in higher-level error handling code, for example, to collect all the information down the error chain to present it to the end user or the calling system. Each error, in this case, will be hidden behind a trait object of a [`Fail`](https://docs.rs/failure/0.1.1/failure/trait.Fail.html) trait. However, there is not much you can do with that `Fail` trait. Basically, you can go down the chain or convert an error to its string representation.

However, to help with that, `Fail` trait has a built-in downcasting functionality. There is a function on `Fail` trait, `downcast_ref` which, given the target type, tries to "cast" the referenced `Fail` implementor to that type. If type matches (for example, you have `MyError` type implementing `Fail` trait and you are casting `Fail` trait object to that `MyError` type), it will return a reference to the concrete type.

This way you can cast an error to something more concrete and retrieve additional information from it (for example, it could be column and line information).

One limitation, though, is that you can only cast to a concrete type. You cannot cast to another trait. Therefore, if you have different types of errors, but which have similar "extra" information attached to them (like line and column information, or any other additional detailed information about the error), the code which processes the chain of errors, needs to know all the possible types.

Not very convenient! Can we do better?

What if we define our own trait, `ExtraInfo`, like this:

```rust
trait ExtraInfo {
    /// Globally unique error code (like "ERR000243").
    fn code() -> &str;
    /// Location where error happened
    fn location() -> String;
}
```

So we can then somehow "cross-cast" a trait object of `Fail` to a trait object of `ExtraInfo`. Then, we could call `code` and `location` functions on that `ExtraInfo` trait object to collect additional information. Like casting one interface to another in Java!

### Plugin Registry

Another example would be a plugin registry. Let's say, we want to keep a global map of plugins where each plugin is known by its base `Plugin` trait. However, internally, each plugin could implement additional "interfaces" and we want a mechanism where we can query each plugin for additional functionality.

We could have separate maps for different plugin interfaces. Every time we would need, let's say, `TimerPlugin` we could go into `timer_plugins` map and look for a plugin there. However, let's assume we want to have a single place where we register plugins (maybe, we have a mechanism to build this map based on a dynamic configuration of some sort!).

Let's try to build this plugin registry.

## Building the Registry

For this experiment, we will start with two specific plugin interfaces. Both are just regular Rust traits:

```rust
/// Generate a greeting message for the given name.
trait Greeter {
    fn greet(&self, name: &str) -> String;
}

/// This is a formal version, which uses a first name and a last name.
trait FormalGreeter {
    fn greet_formal(&self, first_name: &str, last_name: &str) -> String;
}
```

We could have merged both traits into one, with both functions, `greet` and `greet_formal`. If a certain function is "not supported", we could have used a `Result` returning an error (as one of the options).

It would be a runtime error if we would try to invoke a missing functionality, but it would be the same for the dynamic dispatch: if functionality is not supported, we would only know about that at runtime.

Let's create two implementations of these plugins. One implementation will support both the simple greeting interface and the formal one. Another implementation will count greets and will only support the formal greeting interface[^2].


```rust
use std::sync::atomic::{AtomicUsize, Ordering};

/// Simple greeter, implements both `Greeter` and `FormalGreeter` traits
pub struct SimpleGreeter(String);

impl Greeter for SimpleGreeter {
    fn greet(&self, name: &str) -> String {
        format!("{}, {}!", self.0, name)
    }
}

impl FormalGreeter for SimpleGreeter {
    fn greet_formal(&self, first_name: &str, last_name: &str) -> String {
        format!("{}, {} {}!", self.0, first_name, last_name)
    }
}

/// Counting greeter, only implements `FormalGreeter` trait
pub struct CountingGreeter(AtomicUsize);

impl FormalGreeter for CountingGreeter {
    fn greet_formal(&self, first_name: &str, last_name: &str) -> String {
        self.0.fetch_add(1, Ordering::Relaxed);
        format!("Greetings, {} {}.", last_name, first_name)
    }
}
```

Now we want to implement a function that tries to an informal greeter interface, if it supported by the provided plugin. If plugin, however, does not support an informal interface, it will try to use formal interface. The function will use a dynamic dispatch, since according to the story, we don't know the real implementation until runtime.

```rust
pub fn rsvp(first_name: &str, last_name: &str, plugin: &???) -> String {
    // what do we write here?...
}
```

First, what should we specify as the `plugin` parameter type, so it basically means "any implementation whatsoever"? Like in dynamic typing, there you might have a reference to "any object". [Any](https://doc.rust-lang.org/std/any/index.html) ideas?

### Any

There's a trait in Rust, [`std::any::Any`](https://doc.rust-lang.org/std/any/index.html) which is said to emulate dynamic typing. Let's try to use it here.

As a first step, let's take a look at the `std::any::Any` and see if it can help us in one way or another. Let's try to use it in place of the questions marks above and see if we can use it to downcast to our traits.

```rust
use std::any::Any;

pub fn rsvp(first_name: &str, last_name: &str, plugin: &Any) -> String {
    if let Some(gt) = plugin.downcast_ref::<Greeter>() {
        return gt.greet(first_name);
    }
    if let Some(gt) = plugin.downcast_ref::<FormalGreeter>() {
        return gt.greet_formal(first_name, last_name);
    }
    "Hello?".to_string()
}
```

It doesn't work, however: 

```
error: the `downcast_ref` method cannot be invoked on a trait object
   --> src/lib.rs:162:30
    |
162 |     if let Some(gt) = plugin.downcast_ref::<Greeter>() {
    |                              ^^^^^^^^^^^^
```

It's not surprising, as documentation explicitly says "cannot be used to test whether a type implements a trait". Ok. Let's take a step back here and use the concrete types. After all, we know that only `SimpleGreeter` implements `Greeter` trait and `CountingGreeter` only implements a formal greeting.

Let's quickly replace downcasting to `plugin.downcast_ref::<SimpleGreeter>()` and `plugin.downcast_ref::<CountingGreeter>()`, then run the following test:

```rust
#[test]
fn test() {
    let simple = SimpleGreeter("Hi".to_string());
    let formal = CountingGreeter(Default::default());
    assert_eq!("Hi, Andrew!", rsvp("Andrew", "Baker", &simple));
    assert_eq!("Greetings, Baker Andrew.", rsvp("Andrew", "Baker", &formal));
    assert_eq!(1, formal.0.load(Ordering::Relaxed));
}
```

The test passes. That's encouraging, but that's not exactly what we want -- our function should not know about concrete data types. We are modeling an open plugin registry, after all.

Let's dig deeper into `Any` trait to see how it works and why it doesn't support traits.

### Digging into Any

Let's take a look at the `downcast_ref` function of the `Any` trait, which could be seen [here](https://doc.rust-lang.org/1.26.2/src/core/any.rs.html#196-204).

The `downcast_ref` function on `Any` trait checks if `self` is of type `T` and if so, does an unsafe cast to type `T`. This is a problem number one. In case `T` is a trait, we are supposed to return a trait object. However, trait object needs to have two pointers: pointer to the data itself and pointer to the virtual table. We have a data pointer (via `self`), but we don't have a pointer to the virtual table!

The problem number two is the implementation of the `is` function. It doesn't do much: it gets some magic "type id" value" and compares it to the value reported by the data type itself. There is a `get_type_id` function on an `Any` trait which could be invoked via a trait object. The implementation of this function for each concrete type knows the type id (by using the same `TypeId` machinery).

Can we extend this mechanism to work with traits?

### An Idea

First, let's draft the downcasting function we are looking for. It should be something that is very similar to the `downcast_ref` of `Any`, but should allow for traits in its signature:

```rust
pub fn downcast_ref<T: ?Sized>(&self) -> Option<&T> { ... }
```

The difference here is that we are canceling the `Sized` requirement by using a negative bound `?Sized`, so we can substitute a trait name for the type variable `T`. If `T` is substituted with a trait name, the return type would be an `Option` of trait object.

What about the internal implementation? We need a way to somehow get a pointer to the virtual table. This virtual table must correspond to the implementation of trait substituted for `T`. The idea here is that by using a dynamic dispatch on a common interface `Plugin`, we can "ask" the type itself to provide its implementation for the given trait `T`.

The plan is to create a function on a trait that the given target trait type `T` will return a trait object `&T`. I was about to call this function [`QueryInterface`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682521(v=vs.85).aspx)[^5], but decided to use a different name:

```rust
pub trait Plugin {
    fn __downcast_ref(&self, target: ???) -> Option<???>;
}
```

However, what should we take as the `target` parameter and what should we return from this function? The first answer is straightforward: we can use the same `TypeId` mechanism to get the unique identifier for the trait.

The second one is a little bit tricky. We cannot return a trait object `&T`, as this function cannot have type parameters[^3]. If we cannot return a trait object, can we return... a [`TraitObject`](https://doc.rust-lang.org/1.26.2/std/raw/struct.TraitObject.html)?

`TraitObject` is a type in the standard library which has the same memory representation as a trait object. It is a nightly-only struct, but it can be re-created in user code (with an assumption that layout won't change[^4]). However, for this exercise, we would enable the nightly feature.

So, the `Plugin` trait and its implementation for `SimpleGreeter` will look like this:

```rust
#![feature(raw)]
use std::any::TypeId;
use std::raw::TraitObject;

pub trait Plugin: 'static {
    fn __downcast_ref(&self, _target: TypeId) -> Option<TraitObject> {
        None
    }
}

impl Plugin for SimpleGreeter {
    fn __downcast_ref(&self, target: TypeId) -> Option<TraitObject> {
        unsafe {
            if target == TypeId::of::<Greeter>() {
                return Some(std::mem::transmute(self as &Greeter));
            }
            // other interfaces
            // ...
        }
        None
    }
}
```

In the implementation, we check target trait against all traits implemented by the concrete type. In case there is a match, we generate trait object by casting `self` (which is of `&SimpleGreeter` type) to the reference to the target trait. Then, we use unsafe [`std::mem::transmute`](https://doc.rust-lang.org/std/mem/fn.transmute.html) call to transmute it to the `TraitObject`.

Finally, let's look at the public API for the downcasting, the `downcast_ref` function on the `Plugin` trait. Now that we have a way to retrieve `TraitObject` for the given trait type `T`, we can simply cast it back:

```rust
impl Plugin {
    pub fn downcast_ref<T: ?Sized + 'static>(&self) -> Option<&T> {
        unsafe {
            if let Some(obj) = self.__downcast_ref(TypeId::of::<T>()) {
                Some(*(&obj as *const TraitObject as *const &T))
            } else {
                None
            }
        }
    }
}
```

### Minor Inconvenience

Note that we cannot transmute `TraitObject` directly to the type `&T` using `Some(std::mem::transmute(obj))`, since we would get an error:

```
   |
53 |                 Some(std::mem::transmute(obj))
   |                      ^^^^^^^^^^^^^^^^^^^
   |
   = note: source type: std::raw::TraitObject (128 bits)
   = note: target type: &T (pointer to T)
```

The reason is that we cannot enforce `T` to be substituted with a trait. If it is substituted with the concrete type name, the size of the reference would be the size of one pointer and it would not be possible to transmute. Instead, we cast the pointer to the `TraitObject` into the pointer to `&T` and read the value using that pointer.

### Test!

Finally, let's rewrite our function to use `Plugin` instead of `Any` and change the test code:

```rust
pub fn rsvp(first_name: &str, last_name: &str, plugin: &Plugin) -> String {
    if let Some(gt) = plugin.downcast_ref::<Greeter>() {
        return gt.greet(first_name);
    }
    if let Some(gt) = plugin.downcast_ref::<FormalGreeter>() {
        return gt.greet_formal(first_name, last_name);
    }
    "Hello?".to_string()
}

#[test]
fn test() {
    let simple: &Plugin = &SimpleGreeter("Hi".to_string()) as &Plugin;
    let formal: &Plugin = &CountingGreeter(Default::default()) as &Plugin;
    assert_eq!("Hi, Andrew!", rsvp("Andrew", "Baker", simple));
    assert_eq!("Greetings, Andrew Baker.", rsvp("Andrew", "Baker", formal));
}
```

It compiles and the test passes!

## Few Improvements

### Casting to the Concrete Type

We made our `Plugin` trait to support casting to other traits. However, compared to `Any`, we lost the ability to cast to the concrete type. Can we add it back?

Let's consider the following alteration to the `__downcast_ref` of `SimpleGreeter` (and the similar block of code for the `CountingGreeter`):

```rust
if target == TypeId::of::<SimpleGreeter>() {
    return Some(TraitObject {
        data: self as *const _ as *mut (),
        vtable: std::ptr::null_mut(),
    })
}
```

If the target is the concrete type itself, we are constructing a `TraitObject` with a valid data pointer, but the null pointer for the virtual table. Remember that minor inconvenience with casting `TraitObject` into `&T`? Turns out, the way we did that casting is actually beneficial! If `T` is a concrete type, the reference to `T` would be a size of one pointer. Therefore, we would only read the first field, `data`, from the `TraitObject`, making our casting work for the concrete type itself, too!

Let's add the following piece to the test to see if that works:

```rust
// cast back to the concrete type
let simple = simple.downcast_ref::<SimpleGreeter>().unwrap();
assert_eq!(simple.0, "Hi");

// cast back to the concrete type
let formal = formal.downcast_ref::<CountingGreeter>().unwrap();
assert_eq!(1, formal.0.load(Ordering::Relaxed));
```

And all the tests pass! With that addition, we can now also cast to the concrete data types, too!

### Creating Macro

One unfortunate property of this approach is the need to write this implementation of `__downcast_ref` manually. Let's create a macro to do that:

```rust
#[macro_export]
macro_rules! declare_interfaces (
    ( $typ: ident, [ $( $iface: ident ),* ]) => {
        impl Plugin for $typ {
            fn __downcast_ref(&self, target: ::std::any::TypeId) -> Option<::std::raw::TraitObject> {
                unsafe {
                    $(
                    if target == ::std::any::TypeId::of::<$iface>() {
                        return Some(::std::mem::transmute(self as &$iface));
                    }
                    )*
                }
                if target == ::std::any::TypeId::of::<$typ>() {
                    Some(::std::raw::TraitObject {
                        data: self as *const _ as *mut (),
                        vtable: std::ptr::null_mut(),
                    })
                } else {
                    None
                }
            }
        }
    }
);
```

Now instead of implementing function manually we can do:

```rust
declare_interfaces!(SimpleGreeter, [Greeter, FormalGreeter]);
declare_interfaces!(CountingGreeter, [FormalGreeter]);
```

### Cross-casting

Finally, there is one small annoying thing that it is not possible to cross-cast from one plugin interface to another:

```rust
let simple = simple.downcast_ref::<Greeter>().unwrap();
// Nope! Compilation error!
let simple = simple.downcast_ref::<FormalGreeter>().unwrap();
```

It wouldn't be an issue if it was possible to upcast from a `Greeter` or `FormalGreeter` trait into a `Plugin` trait, but it is not[^6]. This could be solved, though, by adding another trait, `PluginBase` (or, alternatively, could be added to `Plugin` trait and `declare_interfaces!` macro changed to implement it):

```rust
pub trait PluginBase {
    fn as_plugin(&self) -> &(Plugin + 'static);
}

impl<T: Plugin> PluginBase for T {
    fn as_plugin(&self) -> &Plugin {
        self
    }
}
```

Then making `Plugin` to require `PluginBase` (so every `&Plugin` get `as_plugin` function):

```rust
pub trait Plugin: PluginBase + 'static { ... }
```

Finally, making each plugin trait to require `Plugin` bound (which wasn't required before, but if we want to get that `as_plugin` function, we have to do it): 

```rust
trait Greeter: Plugin { ... }
```

## Final Thoughts

So, this is one way you can do casting between traits! One limitation, however, limited in that it requires a `'static` bound on the type.

Another limitation is that all "implemented" interfaces need to be declared upfront (because in this implementation we need to generate a "dispatching" function that should know all supported interfaces). This shortcoming could be lifted by implementing a dynamic registry for the implemented traits. In the end, all we need is a mapping from two type identifiers (`TypeId` of the concrete type and `TypeId` of a trait) into a virtual table, corresponding to that combination. It could be a shared map that is somehow dynamically built and then looked up during the casting. Initializing such map, however, could be tricky -- the beauty of the implementation above is that it does not require any initialization.

The full source code is [here](https://github.com/idubrov/experiments/blob/master/casting-traits/src/lib.rs).

### References

[^1]: Full disclaimer: I did use Java a lot, so be warned!
[^2]: I'm using an `AtomicUsize` for the interior mutability to avoid dealing with mutable references (all references to trait objects will be shared references).
[^3]: [Object Safety Is Required for Trait Objects](https://doc.rust-lang.org/1.26.2/book/second-edition/ch17-02-trait-objects.html#object-safety-is-required-for-trait-objects)
[^4]: [It might, though](https://github.com/rust-lang/rfcs/issues/2035)
[^5]: This is indeed a reference to the [Component Object Model](https://en.wikipedia.org/wiki/Component_Object_Model) which works similarly (the way I remember it)!
[^6]: [Why doesn't Rust support trait object upcasting?](https://stackoverflow.com/questions/28632968/why-doesnt-rust-support-trait-object-upcasting)
