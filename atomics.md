Atomic operations are the main building block for anything involving multiple threads. All the other concurrency primitives, such as mutexes and condition variables, are implemented using atomic operations.

In Rust, atomic operations are available as methods on the standard atomic types that live in std::sync::atomic. They all have names starting with Atomic, such as AtomicI32 or AtomicUsize. Which ones are available depends on the hardware architecture and sometimes operating system, but almost all platforms provide at least all atomic types up to the size of a pointer.

Unlike most types, they allow modification through a shared reference (e.g., &AtomicU8). This is possible thanks to interior mutability,

Each of the available atomic types has the same interface with methods for storing and loading, methods for atomic "fetch-and-modify" operations, and some more advanced "compare-and-exchange" methods.

Every atomic operation takes an argument of type std::sync::atomic::Ordering, which determines what guarantees we get about the relative ordering of operations. The simplest variant with the fewest guarantees is Relaxed. Relaxed still guarantees consistency on a single atomic variable, but does not promise anything about the relative order of operations between different variables.

What this means is that two threads might see operations on different variables happen in a different order. For example, if one thread writes to one variable first and then to a second variable very quickly afterwards, another thread might see that happen in the opposite order.

# Atomic Load and Store Operations

The first two atomic operations we’ll look at are the most basic ones: load and store. Their function signatures are as follows, using AtomicI32 as an example:

```
impl AtomicI32 {
    pub fn load(&self, ordering: Ordering) -> i32;
    pub fn store(&self, value: i32, ordering: Ordering);
}

```

The load method atomically loads the value stored in the atomic variable, and the store method atomically stores a new value in it. Note how the store method takes a shared reference (&T) rather than an exclusive reference (&mut T), even though it modifies the value.

# Fetch-and-Modify Operations

Now that we’ve seen a few use cases for the basic load and store operations, let’s move on to more interesting operations: the fetch-and-modify operations. These operations modify the atomic variable, but also load (fetch) the original value, as a single atomic operation.

Note that the atomic types do not implement the Copy trait, so we’d have gotten an error if we had tried to move one into more than one thread.

# Compare-and-Exchange Operations
The most advanced and flexible atomic operation is the compare-and-exchange operation. This operation checks if the atomic value is equal to a given value, and only if that is the case does it replace it with a new value, all atomically as a single operation. It will return the previous value and tell us whether it replaced it or not.

```
fn increment(a: &AtomicU32) {
    let mut current = a.load(Relaxed); 1
    loop {
        let new = current + 1; 2
        match a.compare_exchange(current, new, Relaxed, Relaxed) { 3
            Ok(_) => return, 4
            Err(v) => current = v, 5
        }
    }
}

```

Next to compare_exchange, there is a similar method named compare_exchange_weak. The difference is that the weak version may still sometimes leave the value untouched and return an Err, even though the atomic value matched the expected value. On some platforms, this method can be implemented more efficiently and should be preferred in cases where the consequence of a spurious compare-and-exchange failure are insignificant, such as in our increment function above

```
fn allocate_new_id() -> u32 {
    static NEXT_ID: AtomicU32 = AtomicU32::new(0);
    let mut id = NEXT_ID.load(Relaxed);
    loop {
        assert!(id < 1000, "too many IDs!");
        match NEXT_ID.compare_exchange_weak(id, id + 1, Relaxed, Relaxed) {
            Ok(_) => return id,
            Err(v) => id = v,
        }
    }
}
```

The atomic types have a convenience method called fetch_update for the compare-and-exchange loop pattern. It’s equivalent to a load operation followed by a loop that repeats a calculation and compare_exchange_weak, just like what we did above.

If the atomic variable changes from some value A to B and then back to A after the load operation, but before the compare_exchange operation, it would still succeed, even though the atomic variable was changed (and changed back) in the meantime. In many cases, as with our increment example, this is not a problem. However, there are certain algorithms, often involving atomic pointers, for which this can be a problem. This is known as the ABA problem.


