we saw the std::sync::Arc<T> type that allows for shared ownership through reference counting. The Arc::new function creates a new allocation, just like Box::new. However, unlike a Box, a cloned Arc will share the original allocation, without creating a new one. The shared allocation will only be dropped once the Arc and all its clones are dropped.

The memory ordering considerations involved in an implementation of this type can get quite interesting. In this chapter, we’ll put more of the theory to practice by implementing our own Arc<T>. We’ll start with a basic version, then extend it to support weak pointers for cyclic structures, and finish the chapter with an optimized version that’s nearly identical to the implementation in the standard library.

# Basic Reference Counting
Our first version will use a single AtomicUsize to count the number of Arc objects that share an allocation. Let’s start with a struct that holds this counter and the T object:

```
struct ArcData<T> {
    ref_count: AtomicUsize,
    data: T,
}
```
Note that this struct is not public. It’s an internal implementation detail of our Arc implementation.

Next is the Arc<T> struct itself, which is effectively just a pointer to a (shared) ArcData<T> object.

It might be tempting to make it a wrapper for a Box<ArcData<T>>, using a standard Box to handle the allocation of the ArcData<T>. However, a Box represents exclusive ownership, not shared ownership. We can’t use a reference either, because we’re not just borrowing the data owned by something else, and its lifetime ("until the last clone of this Arc is dropped") is not directly representable with a Rust lifetime.

Instead, we’ll have to resort to using a pointer and handle allocation and the concept of ownership manually. Instead of a *mut T or *const T, we’ll use a std::ptr::NonNull<T>, which represents a pointer to T that is never null. That way, an Option<Arc<T>> will be the same size as an Arc<T>, using the null pointer representation for None.

Plot No.&nbsp;B-182,&nbsp;Sector-2A- Pocket-B
devsuccess@turing.com


IT-related queries (Slack, Google, etc.): turingitsupport@turing.com 

Turing University - Docebo support (same platform): talentdevelopment@turing.com 

Operational inquiries (Jibble, pay, or others): devsuccess@turing.com