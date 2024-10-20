1. Operating systems isolate processes from each other as much as possible, allowing a program to do its thing while completely unaware of what any other processes are doing. For example, a process cannot normally access the memory of another process, or communicate with it in any way, without asking the operating system’s kernel first.
 However, a program can spawn extra threads of execution, as part of the same process. Threads within the same process are not isolated from each other. Threads share memory and can interact with each other through that memory.
 New threads are spawned using the std::thread::spawn function from the standard library which takes a function as argument which thread will execute 
 The Rust standard library assigns every thread a unique identifier. This identifier is accessible through Thread::id() and is of the type ThreadId.
 The .join() method waits until the thread has finished executing , just call .unwrap() to panic when joining a panicked thread.
 The println macro uses std::io::Stdout::lock() to make sure its output does not get interrupted. A println!() expression will wait until any concurrently running one is finished before writing any output.
 Rather than passing the name of a function to std::thread::spawn, as in our example above, it’s far more common to pass it a closure. This allows us to capture values to move into the new thread.
 The spawn function has a 'static lifetime bound on its argument type.It means, it only accepts functions that may be kept around forever. A closure capturing a local variable by reference may not be kept around forever, since that reference would become invalid the moment the local variable ceases to exist.
 The ::spawn fucntion is just a convenient shorthand for std::thread::Builder::new().spawn().unwrap(). A std::thread::Builder allows you to set some settings for the new thread before spawning it. You can use it to configure the stack size for the new thread and to give the new thread a name. The name of a thread is available  through std::thread::current().name(). It will be used in panic messages, and will be visible in monitoring and debugging tools on most platforms.
 Additionally, Builder's spawn function returns an std::io::Result, allowing you to handle situations where spawning a new thread fails.This might happen if the operating system runs out of memory, or if resource limits have been applied to your program. The std::thread::spawn function simply panics if it is unable to spawn a new thread as its a unwrap on builder 
 The Rust standard library provides the std::thread::scope function to spawn  scoped threads. It allows us to spawn threads that cannot outlive the scope of the closure we pass to that function, making it possible to safely borrow local variables.
 There are several ways to create something that’s not owned by a single thread - two are `Statics` and `Leaking`
 #Statics
``` 
static X: [i32; 3] = [1, 2, 3];

thread::spawn(|| dbg!(&X));
thread::spawn(|| dbg!(&X));

```
 Every thread can borrow it, since it’s guaranteed to always exist.

#Leaking
Another way to share ownership is by leaking an allocation. Using Box::leak, we can release ownership of our program, promising to never drop it From that point on, the Box will live forever, without an owner, allowing it to be borrowed by any thread for as long as the program runs
```
let x: &'static [i32; 3] = Box::leak(Box::new([1, 2, 3]));

thread::spawn(move || dbg!(x));
thread::spawn(move || dbg!(x));
```
The move closure might make it look like we’re moving ownership into the threads, but a closer look at the type of x reveals that we’re only giving the threads a reference to the data.

Note how the 'static lifetime doesn’t mean that the value lived since the start of the program, but only that it lives to the end of the program. The past is simply not relevant.

The downside of leaking a Box is that we’re leaking memory. We allocate something, but never drop and deallocate it.

#Reference Counting
To make sure shared data get dropped and deallocated , we can not completely give up its ownership but we can share ownership.By keeping track of the number of owners, we can make sure the value is dropped only when there are no owners left.The Rust standard library provides this functionality through the std::rc::Rc type, short for "reference counted." It is very similar to a Box, except cloning it will not allocate anything new, but instead increment a counter stored next to the contained value. Both the original and cloned Rc will refer to the same allocation; they share ownership.
Dropping an Rc will decrement the counter. Only the last Rc, which will see the counter drop to zero, will be the one dropping and deallocating the contained data.
Rc is not thread safe , If multiple threads had an Rc to the same allocation, they might try to modify the reference counter at the same time, which can give unpredictable results.Instead, we can use std::sync::Arc, which stands for "atomically reference counted." It’s identical to Rc, except it guarantees that modifications to the reference counter are indivisible atomic operations, making it safe to use it with multiple threads.
Because ownership is shared, reference counting pointers (Rc<T> and Arc<T>) have the same restrictions as shared references (&T).They do not give you mutable access to their contained value, since the value might be borrowed by other code at the same time.

# Super good example , feels so good
```
fn f(a: &i32, b: &mut i32) {
    let before = *a;
    *b += 1;
    let after = *a;
    if before != after {
        x(); // never happens
    }
}
```
Here, we get an immutable reference to an integer, and store the value of the integer both before and after incrementing the integer that b refers to. The compiler is free to assume that the fundamental rules about borrowing and data races are upheld, which means that b can’t possibly refer to the same integer as a does. In fact, nothing in the entire program can mutably borrow the integer that a refers to as long as a is borrowing it. Therefore, the compiler can easily conclude that *a will not change and the condition of the if statement will never be true, and can completely remove the call to x from the program as an optimization.


In Rust, it’s only possible to break any of compiler rules when using unsafe code. "Unsafe" doesn’t mean that the code is incorrect or never safe to use, but rather that the compiler is not validating for you that the code is safe. If the code does violate these rules, it is called unsound.The compiler is allowed to assume, without checking, that these rules are never broken. When broken, this results in something called undefined behavior, which we need to avoid at all costs.

 It might end up executing some entirely unrelated part of the program. It can cause all kinds of havoc. Perhaps surprisingly, undefined behavior can even "travel back in time," causing problems in code that precedes it. To understand how that can happen, imagine we had a match statement before our previous snippet, as follows:
 ```
 match index {
   0 => x(),
   1 => y(),
   _ => z(index),
}

let a = [123, 456, 789];
let b = unsafe { a.get_unchecked(index) };
```
Because of the unsafe code, the compiler is allowed to assume index is only ever 0, 1, or 2. It may logically conclude that the last arm of our match statement will only ever match a 2, and thus that z is only ever called as z(2). That conclusion might be used not only to optimize the match, but also to optimize z itself. This can include throwing out unused parts of the code.
If we execute this with an index of 3, our program might attempt to execute parts that have been optimized away, resulting in completely unpredictable behavior, long before we get to the unsafe block on the last line. Just like that, undefined behavior can propagate through a whole program, both backwards and forwards, in often very unexpected ways.
When calling any unsafe function, read its documentation carefully and make sure you fully understand its safety requirements: the assumptions you need to uphold, as the caller, to avoid undefined behavior.

#Interior Mutability

A data type with interior mutability slightly bends the borrowing rules. Under certain conditions, those types can allow mutation through an "immutable" reference.we’ve already seen one subtle example involving interior mutability. Both Rc and Arc mutate a reference counter, even though there might be multiple clones all using the same reference counter.As soon as interior mutable types are involved, calling a reference "immutable" or "mutable" becomes confusing and inaccurate, since some things can be mutated through both. The more accurate terms are "shared" and "exclusive"
Keep in mind that interior mutability only bends the rules of shared borrowing to allow mutation when shared. It does not change anything about exclusive borrowing. Exclusive borrowing still guarantees that there are no other active borrows. Unsafe code that results in more than one active exclusive reference to something always invokes undefined behavior, regardless of interior mutability.

Let’s take a look at a few types with interior mutability and how they can allow mutation through shared references without causing undefined behavior.

#Cell

A std::cell::Cell<T> simply wraps a T, but allows mutations through a shared reference. To avoid undefined behavior, it only allows you to copy the value out (if T is Copy), or replace it with another value as a whole. In addition, it can only be used within a single thread.

For above very good example 
```
use std::cell::Cell;

fn f(a: &Cell<i32>, b: &Cell<i32>) {
    let before = a.get();
    b.set(b.get() + 1);
    let after = a.get();
    if before != after {
        x(); // might happen
    }
}
```
Now it can not assume this if condition to be always false as it can be possible both points to same value . It may still assume, however, that no other threads are accessing the cells concurrently.
The restrictions on a Cell are not always easy to work with. Since it can’t directly let us borrow the value it holds, we need to move a value out (leaving something in its place), modify it, then put it back, to mutate its contents:

```
fn f(v: &Cell<Vec<i32>>) {
    let mut v2 = v.take(); // Replaces the contents of the Cell with an empty Vec
    v2.push(1);
    v.set(v2); // Put the modified Vec back
}

```

#RefCell
 Unlike Cell , RefCell allows you to borrow content at a small runtime cost . A RefCell<T> does not only hold a T, but also holds a counter that keeps track of any outstanding borrows.  If you try to borrow it while it is already mutably borrowed (or vice-versa), it will panic, which avoids undefined behavior. Just like a Cell, a RefCell can only be used within a single thread.

Borrowing the contents of RefCell is done by calling borrow or borrow_mut:

use std::cell::RefCell;

fn f(v: &RefCell<Vec<i32>>) {
    v.borrow_mut().push(1); // We can modify the `Vec` directly.
}

While Cell and RefCell can be very useful, they become rather useless when we need to do something with multiple threads. So let’s move on to the types that are relevant for concurrency.


An RwLock or reader-writer lock is the concurrent version of a RefCell. An RwLock<T> holds a T and tracks any outstanding borrows. However, unlike a RefCell, it does not panic on conflicting borrows. Instead, it blocks the current thread—​putting it to sleep.
Borrowing the contents of an RwLock is called locking. By locking it we temporarily block concurrent conflicting borrows, allowing us to borrow it without causing data races.

A Mutex is very similar, but conceptually slightly simpler. Instead of keeping track of the number of shared and exclusive borrows like an RwLock, it only allows exclusive borrows.

#Atomics 

The atomic types represent the concurrent version of a Cell. Like a Cell, they avoid undefined behavior by making us copy values in and out as a whole, without letting us borrow the contents directly.
Unlike a Cell, though, they cannot be of arbitrary size. Because of this, there is no generic Atomic<T> type for any T, but there are only specific atomic types such as AtomicU32 and AtomicPtr<T>. Which ones are available depends on the platform, since they require support from the processor to avoid data races. Since they are so limited in size, atomics often don’t directly contain the information that needs to be shared between threads. Instead, they are often used as a tool to make it possible to share other—​often bigger—​things between threads. When atomics are used to say something about other data, things can get surprisingly complicated.

#UnsafeCell
An UnsafeCell is the primitive building block for interior mutability.
An UnsafeCell<T> wraps a T, but does not come with any conditions or restrictions to avoid undefined behavior. Instead, its get() method just gives a raw pointer to the value it wraps, which can only be meaningfully used in unsafe blocks. It leaves it up to the user to use it in a way that does not cause any undefined behavior.

Most commonly, an UnsafeCell is not used directly, but wrapped in another type that provides safety through a limited interface, such as Cell or Mutex. All types with interior mutability—​including all types discussed above—​are built on top of UnsafeCell.

# Thread Safety: Send and Sync
The language uses two special traits to keep track of which types can be safely used across threads:

A type is Send if it can be sent to another thread. In other words, if ownership of a value of that type can be transferred to another thread. For example, Arc<i32> is Send, but Rc<i32> is not.


A type is Sync if it can be shared with another thread. In other words, a type T is Sync if and only if a shared reference to that type, &T, is Send. For example, an i32 is Sync, but a Cell<i32> is not. (A Cell<i32> is Send, however.)

All primitive types such as i32, bool, and str are both Send and Sync.

Both of these traits are auto traits, which means that they are automatically implemented for your types based on their fields. A struct with fields that are all Send and Sync, is itself also Send and Sync.

The way to opt out of either of these is to add a field to your type that does not implement the trait. For that purpose, the special std::marker::PhantomData<T> type often comes in handy. That type is treated by the compiler as a T, except it doesn’t actually exist at runtime. It’s a zero-sized type, taking no space.

```
use std::marker::PhantomData;

struct X {
    handle: i32,
    _not_sync: PhantomData<Cell<()>>,
}
```
In this example, X would be both Send and Sync if handle was its only field. However, we added a zero-sized PhantomData<Cell<()>> field, which is treated as if it were a Cell<()>. Since a Cell<()> is not Sync, neither is X. It is still Send, however, since all its fields implement Send.

Raw pointers (*const T and *mut T) are neither Send nor Sync, since the compiler doesn’t know much about what they represent.

The way to opt in to either of the traits is the same as with any other trait; use an impl block to implement the trait for your type:

```
struct X {
    p: *mut i32,
}

unsafe impl Send for X {}
unsafe impl Sync for X {}
```

#Locking: Mutexes and RwLocks

 Unlocking is only possible on a locked mutex, and should be done by the same thread that locked it.Protecting data with a mutex is simply the agreement between all threads that they will only access the data when they have the mutex locked. That way, no two threads can ever access that data concurrently and cause a data race.
 Rust standard library provides this functionality through std::sync::Mutex<T>.

 To ensure a locked mutex can only be unlocked by the thread that locked it, it does not have an unlock() method. Instead, its lock() method returns a special type called a MutexGuard. This guard represents the guarantee that we have locked the mutex. It behaves like an exclusive reference through the DerefMut trait, giving us exclusive access to the data the mutex protects. Unlocking the mutex is done by dropping the guard. When we drop the guard, we give up our ability to access the data, and the Drop implementation of the guard will unlock the mutex.

 Very good example 
 ```
 use std::time::Duration;

fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
                thread::sleep(Duration::from_secs(1)); // New!
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}

```
When you run the program now, you will see that it takes about 10 seconds to complete. Each thread only waits for one second, but the mutex ensures that only one thread at a time can do so.

If we drop the guard—and therefore unlock the mutex—before sleeping one second, we will see it happen in parallel instead:

```
fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
                drop(guard); // New: drop the guard before sleeping!
                thread::sleep(Duration::from_secs(1));
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}
```

With this change, this program takes only about one second, since now the 10 threads can execute their one-second sleep at the same time. This shows the importance of keeping the amount of time a mutex is locked as short as possible. Keeping a mutex locked longer than necessary can completely nullify any benefits of parallelism, effectively forcing everything to happen serially instead.

# Lock Poisoning

The unwrap() calls in the examples above relate to lock poisoning.
A Mutex in Rust gets marked as poisoned when a thread panics while holding the lock.When that happens, the Mutex will no longer be locked, but calling its lock method will result in an Err to indicate it has been poisoned.

Calling lock() on a poisoned mutex still locks the mutex. The Err returned by lock() contains the MutexGuard, allowing us to correct an inconsistent state if necessary.

local variables are dropped at the end of the scope they are defined in.

```
if let Some(item) = list.lock().unwrap().pop() {
    process_item(item);
}

```
If our intention was to lock the list, pop an item, unlock the list, and then process the item after the list is unlocked, we made a subtle but important mistake here. The temporary guard is not dropped until the end of the entire if let statement, meaning we needlessly hold on to the lock while processing the item.

But this does not happen for a similar if statement, such as in this example:

```
if list.lock().unwrap().pop() == Some(1) {
    do_something();
}

```
Here, the temporary guard does get dropped before the body of the if statement is executed. The reason is that the condition of a regular if statement is always a plain boolean, which cannot borrow anything. There is no reason to extend the lifetime of temporaries from the condition to the end of the statement.For an if let statement, however, that might not be the case. If we had used front() rather than pop(), for example, item would be borrowing from the list, making it necessary to keep the guard around

#Reader-Writer Lock
A mutex is only concerned with exclusive access. The MutexGuard will provide us an exclusive reference (&mut T) to the protected data even if we only wanted to look at the data and a shared reference (&T) would have sufficed.

A reader-writer lock is a slightly more complicated version of a mutex that understands the difference between exclusive and shared access, and can provide either. It has three states: unlocked, locked by a single writer (for exclusive access), and locked by any number of readers (for shared access). It is commonly used for data that is often read by multiple threads, but only updated once in a while.

The Rust standard library provides this lock through the std::sync::RwLock<T> type. It works similarly to the standard Mutex, except its interface is mostly split in two parts.Instead of a single lock() method, it has a read() and write() method for locking as either a reader or a writer.

It comes with two guard types, one for readers and one for writers: RwLockReadGuard and RwLockWriteGuard. The former only implements Deref to behave like a shared reference to the protected data, while the latter also implements DerefMut to behave like an exclusive reference.

It is effectively the multi-threaded version of RefCell, dynamically tracking the number of references to ensure the borrow rules are upheld.

Both Mutex<T> and RwLock<T> require T to be Send, because they can be used to send a T to another thread. An RwLock<T> additionally requires T to also implement Sync, because it allows multiple threads to hold a shared reference (&T) to the protected data. (Strictly speaking, you can create a lock for a T that doesn’t fulfill these requirements, but you wouldn’t be able to share it between threads as the lock itself won’t implement Sync.)

The Rust standard library provides only one general purpose RwLock type, but its implementation depends on the operating system. There are many subtle variations between reader-writer lock implementations. Most implementations will block new readers when there is a writer waiting, even when the lock is already read-locked. This is done to prevent writer starvation, a situation where many readers collectively keep the lock from ever unlocking, never allowing any writer to update the data.

The biggest difference is that Rust’s Mutex<T> contains the data it is protecting. In C++, for example, std::mutex does not contain the data it protects, nor does it even know what it is protecting. This means that it is the responsibility of the user to remember which data is protected and by which mutex, and ensure the right mutex is locked every time "protected" data is accessed. This is useful to keep in mind when reading code involving mutexes in other languages, or when communicating with programmers who are not familiar with Rust. A Rust programmer might talk about "the data inside the mutex," or say things like "wrap it in a mutex," which can be confusing to those only familiar with mutexes in other languages.

# Thread Parking
One way to wait for a notification from another thread is called thread parking. A thread can park itself, which puts it to sleep, stopping it from consuming any CPU cycles. Another thread can then unpark the parked thread, waking it up from its nap.

```
use std::collections::VecDeque;

fn main() {
    let queue = Mutex::new(VecDeque::new());

    thread::scope(|s| {
        // Consuming thread
        let t = s.spawn(|| loop {
            let item = queue.lock().unwrap().pop_front();
            if let Some(item) = item {
                dbg!(item);
            } else {
                thread::park();
            }
        });

        // Producing thread
        for i in 0.. {
            queue.lock().unwrap().push_back(i);
            t.thread().unpark();
            thread::sleep(Duration::from_secs(1));
        }
    });
}
```

Let’s dive into an example that uses a mutex to share a queue between two threads. In the above example, a newly spawned thread will consume items from the queue, while the main thread will insert a new item into the queue every second. Thread parking is used to make the consuming thread wait when the queue is empty.


However, unpark requests don’t stack up. Calling unpark() two times and then calling park() two times afterwards still results in the thread going to sleep. The first park() clears the request and returns directly, but the second one goes to sleep as usual.

This means that in our example above it’s important that we only park the thread if we’ve seen the queue is empty, rather than park it after every processed item. While it’s extremely unlikely to happen in this example because of the huge (one second) sleep, it’s possible for multiple unpark() calls to wake up only a single park() call.

Unfortunately, this does mean that if unpark() is called right after park() returns, but before the queue gets locked and emptied out, the unpark() call was unnecessary but still causes the next park() call to instantly return. This results in the (empty) queue getting locked and unlocked an extra time. While this doesn’t affect the correctness of the program, it does affect its efficiency and performance.

This mechanism works well for simple situations like in our example, but quickly breaks down when things get more complicated. For example, if we had multiple consumer threads taking items from the same queue, the producer thread would have no way of knowing which of the consumers is actually waiting and should be woken up. The producer will have to know exactly when a consumer is waiting, and what condition it is waiting for.

#Condition Variables
Condition variables are a more commonly used option for waiting for something to happen to data protected by a mutex. They have two basic operations: wait and notify. Threads can wait on a condition variable, after which they can be woken up when another thread notifies that same condition variable. Multiple threads can wait on the same condition variable, and notifications can either be sent to one waiting thread, or to all of them.

The Rust standard library provides a condition variable as std::sync::Condvar.  Its wait method takes a MutexGuard that proves we’ve locked the mutex. It first unlocks the mutex and goes to sleep. Later, when woken up, it relocks the mutex and returns a new MutexGuard (which proves that the mutex is locked again).
It has two notify functions: notify_one to wake up just one waiting thread (if any), and notify_all to wake them all up.


```
use std::sync::Condvar;

let queue = Mutex::new(VecDeque::new());
let not_empty = Condvar::new();

thread::scope(|s| {
    s.spawn(|| {
        loop {
            let mut q = queue.lock().unwrap();
            let item = loop {
                if let Some(item) = q.pop_front() {
                    break item;
                } else {
                    q = not_empty.wait(q).unwrap();
                }
            };
            drop(q);
            dbg!(item);
        }
    });

    for i in 0.. {
        queue.lock().unwrap().push_back(i);
        not_empty.notify_one();
        thread::sleep(Duration::from_secs(1));
    }
});

```

#Summary

Data that is Send can be sent to other threads, and data that is Sync can be shared between threads.

Reference counting (Arc) can be used to share ownership to make sure data lives as long as at least one thread is using it.

Scoped threads are useful to limit the lifetime of a thread to allow it to borrow non-'static data, such as local variables.

&T is a shared reference. &mut T is an exclusive reference. Regular types do not allow mutation through a shared reference.

Some types have interior mutability, thanks to UnsafeCell, which allows for mutation through shared references.

Cell and RefCell are the standard types for single-threaded interior mutability. Atomics, Mutex, and RwLock are their multi-threaded equivalents.

Cell and atomics only allow replacing the value as a whole, while RefCell, Mutex, and RwLock allow you to mutate the value directly by dynamically enforcing access rules.

Thread parking can be a convenient way to wait for some condition.

When a condition is about data protected by a Mutex, using a Condvar is more convenient, and can be more efficient, than thread parking.

















