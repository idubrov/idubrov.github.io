---
layout: post
title:  "Tricking the HashMap"
categories: rust
---

Is it possible to find something in a hashmap if the key you are looking for is not exactly the same as the one you put into that hashmap? At first glance, this might not make any sense at all. The whole purpose of a hashmap is to store something under some key and then look it up using the same key. Right?

Not quite! Consider the following example (I'm using `get` plus `unwrap` instead of `[]` to avoid getting into [indexing](https://doc.rust-lang.org/std/ops/trait.Index.html) in Rust):

```rust
let mut map: HashMap<String, String> = HashMap::new();
map.insert("hello".to_string(), "world".to_string());
assert_eq!("world", map.get("hello").unwrap());
```

This might look like a trivial example, but it's not that simple! Notice that the hashmap stores `String` type as a key. However, the key being looked, "hello" is of type `&str`. Not the same thing[^1]! Very convenient, though, as it allows to lookup for an "owned" `String` type using a "borrowed" `&str` type, so heap allocation could be avoided.

### From String to &str

First, let's look closer at the first example to understand why it works. The naive implementation of hashmap would have `get` function implemented as something like:

```rust
pub fn get(&self, k: &K) -> Option<&V>
```

However, with this signature, looking up a map with a `String` key would require a reference to a string, `&String`. This would make hashmaps less convenient to use. Often, all you have is a string slice, `&str`, and creating an owned `String` just to use it as a lookup key and then discard it later would be wasteful. Therefore, Rust hashmap `get` function has a slightly more complex signature:

```rust
pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
where K: Borrow<Q>, Q: Hash + Eq
```

The idea here is that you can use any type as a lookup key, as long as that type could be "borrowed" from the stored key type.

Let's look at the `Borrow` trait:

```rust
pub trait Borrow<Borrowed> where Borrowed: ?Sized
{
    fn borrow(&self) -> &Borrowed;
}
```

The return type of `borrow` function is a reference to the `Borrowed` type. Without getting into a lot of details and explanation, this means you can borrow anything that you can produce a reference to (given your "owned" value). This "anything" should exist somewhere inside the datatype implementing the `Borrow` trait. `str` "exists" inside `String`, so you can borrow `str` from `String` and get the reference to the borrowed type (which is `&str`).

This is not a very accurate definition, but let's go with that for now. More detailed explanation of the `Borrow` trait could be found in [Rust documentation](https://doc.rust-lang.org/std/borrow/trait.Borrow.html).

When hashmap implementation needs to compare two keys (one stored and one used for lookup), it would borrow from the stored key type and compare borrowed reference to the lookup key reference.

### Complex Key Types

What if you need a mapping from a pair of owned strings, `(String, String)`? One way to do it would be to re-create a tuple of strings and use it as a lookup key:

```rust
// Extra memory allocation to create lookup key!
assert!(map.get(&("hello".to_string(), "world".to_string())).unwrap());
```

However, using `&("hello", "world")`, to avoid allocating strings, doesn't work. You cannot borrow `(&str, &str)` from `(String, String)`, because memory layout is different between those two -- the former doesn't "exist" in the latter!

Let's take a look at a more tricky case. What if for some keys in the map there could be another key that is the same, but starts with "_"[^2]?. For example, for the key "hello" there could be another key, "_hello". When checking the hashmap, we sometimes would want to check both keys while only having a lookup key of "hello".

Even though "hello" exists inside "_hello", we cannot just borrow it. First, the borrowing rules are already defined for `String` and `str`. Second, the borrowed value should behave identically to the value it was borrowed from (for example, the hash code should be the same, so the hash table would check the same bucket).

The straightforward way would be to create a new string with "_" prepended to it and then use it for lookup:

```rust
// This is what was given to us!
let key = "hello";
// Allocating a new String to create a lookup key!
assert_eq!("people", map.get(&format!("_{}", key)).unwrap());
```

Can we avoid those allocations, though? And what would be the performance impact, if any?

### Doing it Hard Way

Not everything is lost, however!

If you look at the `Borrow` trait declaration, you would notice that `Borrowed` is marked with a [negative trait `?Sized`](https://doc.rust-lang.org/std/marker/trait.Sized.html). It means that it is possible to substitute a trait for the `Borrowed` type variable! In which case, `borrow` function would return a [trait object](https://doc.rust-lang.org/book/second-edition/ch17-02-trait-objects.html).

The plan here is to allow borrowing a trait object of our custom trait from the key type (`String`), then pass another trait object as a lookup key (but implemented by a completely different data structure).

Here is what it would look like in the code:

```rust
let mut map = HashMap::new();
map.insert("_hello".to_string(), "world".to_string());
// Create a lookup trait object!
let lookup_key = &ShortKey("hello") as &Key;
assert_eq!("world", map.get(lookup_key).unwrap());
```

For this to work, we need a trait `Key`, a struct `ShortKey` and make both `ShortKey` and `String` to implement the same trait `Key`. It would be possible to implement this trait directly by `&str` and avoid having an extra struct `ShortKey`, but that would make `&str` and `String` to behave inconsistently when "borrowed" as `&Key`.

The `ShortKey` is a wrapper type that contains a reference to the string slice:

```rust
pub struct ShortKey<'a>(&'a str);
```

The `Key` trait should be designed in a way that all its implementations could behave identical to the `String` key in the hashmap and at the same time implementing it for the lookup type should not require allocations on the heap. Something like this will do[^3]:

```rust
pub trait Key {
    // Return the tuple of the first byte of a key plus the rest of the key
    fn key(&self) -> (Option<u8>, &[u8]);
}
```

The `String` implementation would chop off the first character and pair it with the rest of the string:

```rust
impl Key for String {
    fn key(&self) -> (Option<u8>, &[u8]) {
        let bytes = self.as_bytes();
        if bytes.is_empty() {
            (None, b"")
        } else {
            // Split the first byte and return a byte slice
            // corresponding to the rest of the string
            (Some(bytes[0]), &bytes[1..])
        }
    }
}
```

The `ShortKey` implementation should behave identically to the same string, prepended with "_":

```rust
impl<'a> Key for ShortKey<'a> {
    fn key(&self) -> (Option<u8>, &[u8]) {
        // Pretend we are a String with "_" being the first character!
        (Some(b'_'), self.0.as_bytes())
    }
}
```

Make `Key` borrowable from a `String`, so we can use `&Key` type as a lookup key in the map[^4]:

```rust
impl<'a> Borrow<Key + 'a> for String {
    fn borrow(&self) -> &(Key + 'a) {
        self
    }
}
```

### Implementing Hash+Eq

We are not done yet. The next part is to make `&Key` to satisfy another requirement on `get` function: `Hash + Eq`. `PartialEq` (a pre-requisite for `Eq`) is pretty straightforward: get the "key" tuple and compare it with "key" from the other trait object:

```rust
impl<'a> Eq for (Key + 'a) {}

impl<'a> PartialEq for (Key + 'a) {
    fn eq(&self, other: &Key) -> bool {
        self.key() == other.key()
    }
}
```

`Hash`, on the other hand, is a little bit more tricky. The reason here is that when you insert your key in the hashmap, it doesn't know anything about your intentions to use a different lookup key. Therefore, it uses `String` hash to determine the hash bucket to put the value into.

This means that `Key` implementation of `Hash` must replicate `String` behavior exactly.

Let's look at the `String` implementation of `Hash` (or, rather, at `str` implementation since `String` delegates to it):

```rust
state.write(self.as_bytes());
state.write_u8(0xff)
```

So, it's just hash of byte content of the string plus a hash of `0xff` byte. So, we need to hash the first byte, then hash the remaining of the bytes plus hash `0xff` at the end:

```rust
impl<'a> Hash for (Key + 'a) {
    fn hash<H: Hasher>(&self, state: &mut H) {
        match self.key() {
            (Some(c), rest) => {
                state.write_u8(c);
                state.write(rest);
                state.write_u8(0xff)
            }
            (None, s) => s.hash(state),
        }
    }
}
```

And we are done! See the full code plus some tests [here](https://github.com/idubrov/experiments/blob/master/tricking-hashmap/src/lib.rs).

Running the test below produces a green light!

```rust
let mut map = HashMap::new();
map.insert("_hello".to_string(), "world".to_string());
let lookup_key = &ShortKey("hello") as &Key;
assert_eq!("world", map.get(lookup_key).unwrap());
```

### Benchmarks!

Finally, let's measure the difference in performance. The benchmark would be to create a map with both non-prefixed keys and their underscore prefixed variant. Then, iterate through the list of non-prefixed keys and lookup the map for the underscored key, sum the values and check it is equal to the expected number (hashmap size, since every value is equal to "1").

Non-optimized, allocating variant (named `with_allocations` in the benchmark results):

```rust
sum += map[&format!("_{}", key)];
```

Optimized, non-allocating variant (`without_allocations`):

```rust
sum += map[&ShortKey(&key) as &Key];
```

However, after experimenting for a while, I realized that `format!` is slower compared to building a string directly (probably, because of reallocation):

```rust
let mut lookup_key = String::with_capacity(key.len() + 1);
lookup_key.push('_');
lookup_key += key;
sum += map[&lookup_key];
```

The results on my machine are the following:

```
test with_allocations       ... bench:     120,398 ns/iter (+/- 9,483)
test with_smart_allocations ... bench:      59,481 ns/iter (+/- 5,291)
test without_allocations    ... bench:      47,737 ns/iter (+/- 5,414)
```

I also measured with different key sizes and the bigger the key size, the smaller the difference (probably, because on large keys hash computation dominates everything else).

The non-allocating version would also put a less pressure on the heap allocator.

Again, the full code is [here](https://github.com/idubrov/experiments/blob/master/tricking-hashmap/src/lib.rs). In the benchmarking code I also count allocations to make sure that non-allocating variant does not allocate on the heap.

### Final Thoughts

In the end, the non-allocating variant is only marginally better from the performance point of view (as measured with built-in benchmarker). Also, real-world application is unlikely to be doing only hashmap lookups.

Nevertheless, it was fun to explore the way `HashMap` works in Rust and it was a great learning experience.

This idea could be applied to other cases where hashmap key type could not be changed (for example, when using [`serde_json::Value`](https://docs.serde.rs/serde_json/value/enum.Value.html)) or is inconvenient otherwise (for example, a tuple of strings).

### References

[^1]: See [str vs String](http://www.ameyalokare.com/rust/2017/10/12/rust-str-vs-String.html) for explanation.
[^2]: If that sounds like a weird thing to want, here is the [rationale](https://www.hl7.org/fhir/json.html#primitive) for that.
[^3]: The reason it uses byte and a byte slice and not character plus string slice is to avoid minor troubles of chopping the first character off the string in a UTF-8 compatible way. Slicing "rest of the string" as `&s[1..]` would panic if the first character in the string is longer than a single byte. Since our "prefix" character fits into one byte, it is more convenient to use byte slices. I measured both variants and even though `char` variant needs to deal with UTF-8, they both perform about the same.
[^4]: This `+ 'a` part is important to satisfy the borrow checker. It makes it possible to borrow an object trait that has references inside it as long as those references do not outlive the `String` we are borrowing it from, during a lookup in the map. See [this explanation](https://stackoverflow.com/questions/26212397/references-to-traits-in-structs).
