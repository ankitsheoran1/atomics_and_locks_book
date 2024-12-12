A processor might determine that two particular consecutive instructions in your program will not affect each other, and execute them out of order, if that is faster for example. While one instruction is briefly blocked on fetching some data from main memory, several of the following instructions might be executed and finished before the first instruction finishes, as long as that wouldn’t change the behavior of your program.Similarly, a compiler might decide to reorder or rewrite parts of your program if it has reason to believe it might result in faster execution. But, again, only if that wouldn’t change the behavior of your program.
The logic for verifying that a specific reordering or other optimization won’t affect the behavior of your program does not take other threads into account.
in other exanple that’s perfectly fine, as the unique references (&mut i32) guarantee that nothing else can possibly access the values, making other threads irrelevant.
The only situation where this is a problem is when mutating data that’s shared between threads. Or, in other words, when working with atomics. This is why we have to explicitly tell the compiler and processor what they can and can’t do with our atomic operations, since their usual logic ignores interactions between threads and might allow for optimizations that do change the result of your program.

```
let x = a.fetch_add(1,
    Dear compiler and processor,
    Feel free to reorder this with operations on b,
    but if there's another thread concurrently executing f,
    please don't reorder this with operations on c!
    Also, processor, don't forget to flush your store buffer!
    If b is zero, though, it doesn't matter.
    In that case, feel free to do whatever is fastest.
    Thanks~ <3
);

```
The available orderings in Rust are:

Relaxed ordering: Ordering::Relaxed

Release and acquire ordering: Ordering::{Release, Acquire, AcqRel}

Sequentially consistent ordering: Ordering::SeqCst
In C++, there is also something called consume ordering, which has been purposely omitted from Rust, but is nonetheless interesting to discuss as well.

# The Memory Model
Rust’s memory model allows for concurrent atomic stores, but considers concurrent non-atomic stores to the same variable to be a data race, resulting in undefined behavior.
On most processor architectures, however, there is actually no difference between an atomic store and a regular non-atomic store, as we’ll see in Chapter 7. One could argue that the memory model is more restrictive than necessary, but these strict rules make it easier to reason about a program, both for the compiler and the programmer, and they leave space for future developments.



#Happens-Before Relationship
The memory model defines the order in which operations happen in terms of happens-before relationships. This means that as an abstract model, it doesn’t talk about machine instructions, caches, buffers, timing, instruction reordering, compiler optimizations, and so on, but instead only defines situations where one thing is guaranteed to happen before another thing, and leaves the order of everything else undefined. The basic happens-before rule is that everything that happens within the same thread happens in order. If a thread is executing f(); g();, then f() happens-before g().

let’s take a look at the following example where we assume a and b are concurrently executed by different threads:
```
static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);

fn a() {
    X.store(10, Relaxed); 1
    Y.store(20, Relaxed); 2
}

fn b() {
    let y = Y.load(Relaxed); 3
    let x = X.load(Relaxed); 4
    println!("{x} {y}");
}

```

the basic happens-before rule is that everything that happens within the same thread happens in order. In this case: 1 happens-before 2, and 3 happens-before 4, as shown in Figure 3-1. Since we use relaxed memory ordering, there are no other happens-before relationships in our example.

If either of a or b completes before the other starts, the output will be 0 0 or 10 20. If a and b run concurrently, it’s easy to see how the output can be 10 0. One way this can happen is if the operations run in this order

More interestingly, the output can also be 0 20, even though there is no possible globally consistent order of the four operations that would result in this outcome. When 3 is executed, there is no happens-before relationship with 2, which means it could load either 0 or 20. When 4 is executed, there is no happens-before relationship with 1, which means it could load either 0 or 10. Given this, the output 0 20 is a valid outcome.

While atomic operations using relaxed memory ordering do not provide any happens-before relationship, they do guarantee a total modification order of each individual atomic variable. This means that all modifications of the same atomic variable happen in an order that is the same from the perspective of every single thread.

```
static X: AtomicI32 = AtomicI32::new(0);

fn a() {
    X.fetch_add(5, Relaxed);
    X.fetch_add(10, Relaxed);
}

fn b() {
    let a = X.load(Relaxed);
    let b = X.load(Relaxed);
    let c = X.load(Relaxed);
    let d = X.load(Relaxed);
    println!("{a} {b} {c} {d}");
}
```

In this example, only one thread modifies X, which makes it easy to see that there’s only one possible order of modification of X: 0→5→15. It starts at zero, then becomes five, and is finally changed to fifteen. Threads cannot observe any values from X that are inconsistent with this total modification order. This means that "0 0 0 0", "0 0 5 15", and "0 15 15 15" are some of the possible results from the print statement in the other thread, while an output of "0 5 0 15" or "0 0 10 15" is impossible.

Even if there’s more than one possible order of modification for an atomic variable, all threads will agree on a single order.

# Release and Acquire Ordering
Release and acquire memory ordering are used in a pair to form a happens-before relationship between threads. Release memory ordering applies to store operations, while Acquire memory ordering applies to load operations.

A happens-before relationship is formed when an acquire-load operation observes the result of a release-store operation. In this case, the store and everything before it, happened before the load and everything after it.

When using Acquire for a fetch-and-modify or compare-and-exchange operation, it applies only to the part of the operation that loads the value. Similarly, Release applies only to the store part of an operation. AcqRel is used to represent the combination of Acquire and Release, which causes both the load to use acquire ordering, and the store to use release ordering.

```
use std::sync::atomic::Ordering::{Acquire, Release};

static DATA: AtomicU64 = AtomicU64::new(0);
static READY: AtomicBool = AtomicBool::new(false);

fn main() {
    thread::spawn(|| {
        DATA.store(123, Relaxed);
        READY.store(true, Release); // Everything from before this store ..
    });
    while !READY.load(Acquire) { // .. is visible after this loads `true`.
        thread::sleep(Duration::from_millis(100));
        println!("waiting...");
    }
    println!("{}", DATA.load(Relaxed));
}
```

When the spawned thread is done storing the data, it uses a release-store to set the READY flag to true. When the main thread observes this through its acquire-load operation, a happens-before relationship is established between those two operations, as shown in Figure 3-3. At that point, we know for sure that everything that happened before the release-store to READY is visible to everything that happens after the acquire-load. Specifically, when the main thread loads from DATA, we know for sure it will load the value stored by the background thread. There’s only one possible outcome this program can print on its last line: 123.

If we had used relaxed memory ordering for all operations in this example, the main thread could have seen READY flip to true, while still loading a zero from DATA afterwards.

The names "release" and "acquire" are based on their most basic use case: one thread releases data by atomically storing some value to an atomic variable, and another thread acquires it by atomically loading that value. This is exactly what happens when we unlock (release) a mutex and subsequently lock (acquire) it on another thread.


In our example, the happens-before relationship from the READY flag guarantees that the store and load operations of DATA cannot happen concurrently. This means that we don’t actually need those operations to be atomic.

However, if we simply try to use a regular non-atomic type for our data variable, the compiler will refuse our program, since Rust’s type system doesn’t allow us to mutate those from one thread when another thread is also borrowing them. The type system does not magically understand the happens-before relationship we’ve created here. Some unsafe code is necessary to promise to the compiler that we’ve thought about this carefully and we’re sure we’re not breaking any rules, as follows:

```
static mut DATA: u64 = 0;
static READY: AtomicBool = AtomicBool::new(false);

fn main() {
    thread::spawn(|| {
        // Safety: Nothing else is accessing DATA,
        // because we haven't set the READY flag yet.
        unsafe { DATA = 123 };
        READY.store(true, Release); // Everything from before this store ..
    });
    while !READY.load(Acquire) { // .. is visible after this loads `true`.
        thread::sleep(Duration::from_millis(100));
        println!("waiting...");
    }
    // Safety: Nothing is mutating DATA, because READY is set.
    println!("{}", unsafe { DATA });
}
```

A happens-before relationship is formed when an acquire-load operation observes the result of a release-store operation. But what does that mean?
The ordered list of all modifications that happen to an atomic variable. Even if the same value is written to the same variable more than once, each of these operations represents a separate event in the total modification order of that variable. When we load a value, the value loaded matches a specific point on this per-variable "timeline," which tells us which operation we might be synchronizing with.
However, if we have previously (in terms of happens-before relationships) seen a 6, we know we’re seeing the last 7, not the first one, meaning we now have a happens-before relationship with thread one, and not with thread two.

There is one extra detail, which is that a release-stored value may be modified by any number of fetch-and-modify and compare-and-exchange operations, while still resulting in a happens-before relationship with an acquire-load that reads the final result.

Mutexes are the most common use case for release and acquire ordering . When locking, they use an atomic operation to check if it was unlocked, using acquire ordering, while also (atomically) changing the state to "locked." When unlocking, they set the state back to "unlocked" using release ordering. This means that there will be a happens-before relationship between unlocking a mutex and subsequently locking it.

```
static mut DATA: String = String::new();
static LOCKED: AtomicBool = AtomicBool::new(false);

fn f() {
    if LOCKED.compare_exchange(false, true, Acquire, Relaxed).is_ok() {
        // Safety: We hold the exclusive lock, so nothing else is accessing DATA.
        unsafe { DATA.push('!') };
        LOCKED.store(false, Release);
    }
}

fn main() {
    thread::scope(|s| {
        for _ in 0..100 {
            s.spawn(f);
        }
    });
}
```


As we’ve briefly seen in "Compare-and-Exchange Operations" in Chapter 2, compare-and-exchange operations take two memory ordering arguments: one for the case where the comparison succeeded and the store happened, and one for the case where the comparison failed and the store did not happen. In f, we attempt to change LOCKED from false to true, and only access DATA if that succeeds. So, we only care about the success memory ordering. If the compare_exchange operation fails, that must be because LOCKED was already set to true, in which case f doesn’t do anything. This matches the try_lock operation on a regular mutex.

```
impl AtomicI32 {
    pub fn compare_exchange(&self, expected: i32, new: i32) -> Result<i32, i32> {
        // In reality, the load, comparison and store,
        // all happen as a single atomic operation.
        let v = self.load();
        if v == expected {
            // Value is as expected.
            // Replace it and report success.
            self.store(new);
            Ok(v)
        } else {
            // The value was not as expected.
            // Leave it untouched and report failure.
            Err(v)
        }
    }
}



static mut DATA: String = String::new();
static LOCKED: AtomicBool = AtomicBool::new(false);

fn f() {
    if LOCKED.compare_exchange(false, true, Acquire, Relaxed).is_ok() {
        // Safety: We hold the exclusive lock, so nothing else is accessing DATA.
        unsafe { DATA.push('!') };
        LOCKED.store(false, Release);
    }
}

fn main() {
    thread::scope(|s| {
        for _ in 0..100 {
            s.spawn(f);
        }
    });
}
```


compare-and-exchange operations take two memory ordering arguments: one for the case where the comparison succeeded and the store happened, and one for the case where the comparison failed and the store did not happen. In f, we attempt to change LOCKED from false to true, and only access DATA if that succeeds. So, we only care about the success memory ordering. If the compare_exchange operation fails, that must be because LOCKED was already set to true, in which case f doesn’t do anything. This matches the try_lock operation on a regular mutex.


As the fundamental theorem of software engineering tells us, every problem in computer science can be solved by adding another layer of indirection, and this problem is no different. Since we can’t fit the data into a single atomic variable, we can instead use an atomic variable to store a pointer to the data.


# Consume Ordering
Let’s take a closer look at the memory ordering in our last example. If we leave the strict memory model aside and think of it in more practical terms, we could say that the release ordering prevents the initialization of the data from being reordered with the store operation that shares the pointer with the other threads. This is important, since otherwise other threads might be able to see the data before it’s fully initialized.

Consume ordering is basically a lightweight—​more efficient—​variant of acquire ordering, whose synchronizing effects are limited to things that depend on the loaded value.

What that means is that if you consume-load a release-stored value x from an atomic variable, then, basically, that store happened before the evaluation of dependent expressions like *x, array[x] or table.lookup(x + 1), but not necessarily before independent operations, like reading another variable that we don’t need the value of x for.

The good news is that—​on all modern processor architectures—​consume ordering is achieved with the exact same instructions as relaxed ordering. In other words, consume ordering can be "free," which—​at least on some platforms—​is not the case for acquire ordering.


# Sequentially Consistent Ordering

The strongest memory ordering is sequentially consistent ordering: Ordering::SeqCst. It includes all the guarantees of acquire ordering (for loads) and release ordering (for stores), and also guarantees a globally consistent order of operations.


Within a single thread, there is a happens-before relationship between every single operation.


Spawning a thread happens-before everything the spawned thread does.

Everything a thread does happens-before joining that thread.

Unlocking a mutex happens-before locking that mutex again.

A consume-load would be a lightweight version of an acquire-load, if it existed.

Sequentially consistent ordering results in a globally consistent order of operations, but is almost never necessary and can make code review more complicated.

Fences allow you to combine the memory ordering of multiple operations or apply a memory ordering conditionally.

