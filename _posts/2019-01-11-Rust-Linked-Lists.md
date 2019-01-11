---
layout: post
title:  "Designing a Safe Doubly Linked List with C++-like Iterators in Rust (Parts 1 and 2)"
date:   2019-01-11 13:00:00 +0200
categories: Rust
---

This is the follow up post of 16th pl-seminar at National Technical University of Athens, an unofficial conference between programming language researchers, where I demonstrated ways to create safe doubly linked lists and hold persistent iterator-like constructs to them. Boy, was that intense! [Slides are here][slides].

Unfortunately, I could not say everything I wanted in 25 minutes time, and was not prepared to go straight to the point, since most people think different things when talking about linked lists. I probably digressed a lot trying to make my talk more engaging. Learning from my mistakes, I will begin with what I am trying to achieve.

Once upon a time, in the scope of my PhD, I had an (academic) codebase in C++ that used std::list and std::list::iterator. Imagine we have N points on a map and an itinerary of M points:

![Points][points]
{: style="text-align: center;"}

Some specifics of the algorithm:

* In the first phase we have O(N*M) iterations, trying to find the best point and the best position in the itinerary to insert this point. Finally, an insertion is performed at an already found iterator, therefore the operation is O(1)  (You might argue that I do not need a list; you might be correct, and I have yet to benchmark my implementation against a `Vec`-only implementation).
* In the second phase, some elements are going to be deleted from the middle of the list.
* In a variant of the algorithm, I keep specific regions of the list ([begin, end] iterators), and I try to insert each point inside a single region only. After each insertion, the region boundaries might move to contain the newly inserted point.

Some time later, I chose to [RIIR][RIIR]. This would have to be served on the internet, and I was not content with the (then) current kludgy setup, a Ruby [sinatra][sinatra] script that validates the input and communicates with the C++ executable through stdin/stdout. Also, I wanted to learn Rust :)

My story starts when I stumbled upon a problem: Rust [std::collections::LinkedList][std::LinkedList] is garbage for my usecase. It has no C++-like iterators, you cannot insert an element in the middle and the only meaningful operation, splitting, is O(n).

Before delving in the bottomless pit that is singly/doubly linked lists in Rust, I urge you to read the book called "[Learning Rust With Entirely Too Many Linked Lists][Linked List Book]".  This book has a very nice property; if you finish it, you understand Rust. No, seriously. This book is the certain way to understanding the borrow semantics of Rust. In order to understand my post you don’t have to read the full book thoroughly, but I will try my best to not repeat what has already been said. So, I won’t argue why linked lists are bad (in contrast to my talk), but just try to implement one. Therefore, since I won’t repeat most of what my slides say, maybe this is a good time to [actually read them][slides] :)

The first thing we should do before considering to implement our own linked list is to search what others have done, using some DuckDuckGo-fu (appending !g to our query if necessary):

1. [std::collections::LinkedList][std::LinkedList]: I talked about this. O(n) splitting, adding only in the ends. Bad list.
1. [ixlist][ixlist]: This implementation can insert in O(1) anywhere. That sounds promising. It uses a vector as the internal representation. How does it implement insert? It adds to the end of a vector and uses indices as next/prev. But how does it implement delete? It does not implement delete... So, this is not what I need. A for effort though :)
1. [linked-list crate][linked-list crate]: Although this looks perfect, it has a major flaw. You cannot store a cursor for a later time, because each cursor borrows the List mutably. Thus, cursors cannot be persisted, since each list can have a single active cursor, and the list in turn is inaccessible when a cursor is active.

Let's talk about my implementation. We need to fight against the borrowck. My first implementation was a scary beast. I called it SubSequences, it contained multiple linked lists in itself and used a cursor API that was exactly a C++ iterator – a thing that could yield the same value twice, mutably :) Yes, this is undefined behavior in Rust. Yes, my current implementation does not invoke UB. I am a fast learner.

> "Simplicity does not precede complexity, but follows it" - Alan Perlis
    
The trick is that, in order to delete an object, you don't have to remove it from the list. You can have a second vector `reclaims: Vec<usize>` that enumerates all cells that are freed. This is essentially the "free memory" pool, which is prioritized over new cells when an insertion happens. I am definitely not a pioneer in this idea; not even if we narrow our searches to creating linked lists in Rust. For example, this [Reddit thread][reddit thread] does something similar, albeit with a bit more complexity. Also, it argues almost the same problems as me. But let’s take this idea to the extreme anyway, and try to make it general to fit in a library.

I believe I have a better answer in implementing C++-like iterators than most people do. The idea is simple: What is the first advice you are given when you try to store references in Rust (e.g. for self-referential structs)? It is to use an index. So, let's generalize indices! Let's call an index something that:

1. Does not hold a reference to the collection.
1. The collection implements `Index` and `IndexMut` for the index (well, duh)
1. And, the index cannot be incremented or decremented (or bicremented) by itself but needs the collection to increment/decrement it.

As a bonus, we can define a Trait hierarchy similar to the Iterator hierarchy of C++. Bikeshedding pending:

1. An `ImmutableIndex` is a collection that can return indices at the begin and the end of itself, and check for index validity
1. A `ForwardIndex` is a collection that can increment indices
1. A `BackwardIndex` is a collection that can decrement indices
1. A `BidirectionalIndex` is a collection that is `ForwardIndex` + `BackwardIndex`

So, `usize` can be an index for `Vec` and `VecDeque`. The only thing that needs to be done is define the relevant traits for `Vec` and `VecDeque`, as seen in the slides.

The next step is obvious; a `LinkedListIndex` is just an opaque wrapper around the `usize` index of the inner `Vec`. This list follows an index invalidation scheme quite similar to the C++ one: No index will be invalidated unless the actual underlying element is deleted. Also, on par with C++ iterators, there is no check if an index has been invalidated or not. This will be discussed later.

Thus, we have created a way to store the elements' positions without holding a borrow to the collection. These positions won't be invalidated as the collection changes (as long as these elements continue to exist). And we can delete / insert before these positions at a later time.

Another thing to note is that we can now write generic code over all collections that implement these traits. Indices will be slower than Rust Iterators, but can express more things such as going back or resetting the iteration. And in the case of `RandomAccessIndex` (which I have not yet designed) we can define currently slice-only methods such as binary search and quickselect.

There are some concerns regarding the checking of the validity of the indices, but bear in mind that I did not use the word `unsafe` in my train of thoughts, therefore there is no memory safety issue. To put it in Rust terms, it is **perfectly safe** to yield a deleted element from the list.

What is this collection unable to do? Not anything insurmountable.

* Hold datatypes that need to be dropped on deletion. Please don't use it on `MutexGuard` or database connection handlers.
* Fast split or merge. Well, it is feasible, but not easy. Either use the standard library `LinkedList`, or change my implementation so that all lists that will be merged share the same "pool". You might need some Rc/Arc-fu magic.
* Have peak performance. Let's be honest, we bounds-check twice and thrice as much as we would like. But, as I already said, if your algorithm is asymptotically better with lists, try it, and then optimize. Some bounds checks might be skipped by using unsafe (e.g. on iterators) in a more careful implementation :)
* Guarantee that an index won't be used on the wrong collection. With a simple implementation, you may get an index from a collection and use it on another collection. To have better guarantees, you could embed a GUID on each collection and share it with the index upon creation.
* Guarantee that indexes that are tested for equality belong to the same collection. Ditto, a GUID will probably save the day.
* Guarantee an index points to an element that has not been deleted. You need to have an increasing number (serving as a timestamp) that will be increased on each element deletion, and check each access.

There it is.

1. An almost no-unsafe (IterMut has the bad word `unsafe`, for obvious reasons – the compiler cannot possibly reason that it will not yield the same elements)
1. Almost doubly linked list.
1. With C++-like iterators.

The main reason I have not published a crate with this idea is that, in order to be widely available, the index traits (after extreme bikeshedding) have to land on std, or on a widely accepted crate. The list can definitely exist as its own crate, but the indices have to become the de facto standard when implementing a new datatype. Otherwise it will be dependency hell using these traits to write generic code that works with all third party collections (see [Orphan Rule][Rustbook Traits]).

I do not consider the source code of my implementation as part of this blog post, but since I once stripped down an old snapshot of it, I find no harm in [making it public][source code]. It is not commented and many parts such as the `Index` traits have been changed countless times, so I will not make any remarks on it.

---
---

### Part 2

This blog post was supposed to end here. But, since my talk, I had some other thoughts. I considered publishing in two parts, but I am not that well known in the community to do so... Continue after reading the [slides][slides], especially the trait definitions :)

Notice that this `Index` hierarchy does not deter implementation of unsafe lists. The index operation just has to ignore the self argument and simply dereference the inner pointer of the index. So, after giving it some thought, I believe I can give you *full* safe (but not no-`unsafe`) linked lists. Bold statement, right? I would gladly implement this, but I am not yet quite confident neither in unsafe Rust, nor in thinking of all the corner cases.

Let’s implement a datatype that has the following features:

1. O(1) insertion when we have an index there.
1. O(1) splitting and merging when we have the appropriate indices
1. Is a `BidirectionalIndex`
1. And has the most coveted property, safety. We might need a master of the [Coq][Coq] to prove it, but this is a brainstorming post, so allow me to approach it intuitively.

Let’s stop being fuzzy. The following will be mostly Rust (it should compile with minor changes) but needs polishing before it can turn into a library:

```rust
struct ListNode<T>
{
    element: T,
    e_tag: Uuid,
    next: *mut ListNode<T>,
    prev: *mut ListNode<T>
}

struct ListIndex<T>
{
    list_id: Uuid,
    e_tag: Uuid,
    index: *mut ListNode<T>
}

struct List<T>
{
    list_id: Uuid,
    first: *mut ListNode<T>,
    last: *mut ListNode<T>,
    reclaims: Vec<*mut ListNode<T>>
}
```

We have to guarantee memory safety. Scratch that, we are using `unsafe`. We have to guarantee **absence of UB**. Ouch. Let's quote [Rustonomicon][Rustonomicon]:


> Unlike C, Undefined Behavior is pretty limited in scope in Rust. All the core language cares about is preventing the following things:
> 
> * Dereferencing null, dangling, or unaligned pointers
> * Reading uninitialized memory
> * Breaking the pointer aliasing rules
> * Producing invalid primitive values:
>   * dangling/null references
>   * null fn pointers
>   * a bool that isn't 0 or 1
>   * an undefined enum discriminant
>   * a char outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
>   * A non-utf8 str
> * Unwinding into another language
> * Causing a data race

Let's group them into three categories. Memory safety, aliasing compliance and irrelevant. Irrelevant first:

* Producing invalid primitive values:
  * null fn pointers
  * a bool that isn't 0 or 1
  * an undefined enum discriminant
  * a char outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
  * A non-utf8 str
* Unwinding into another language

Unless I am completely mistaken, these cannot happen.

Memory safety:

* Dereferencing null, dangling, or unaligned pointers
* Reading uninitialized memory
* Producing invalid primitive values:
  * dangling/null references

We have to prove two (three) things:

1. No index will be used on a freed element. Easy, *never* free an element as long as the list is alive (using the vector of reusable pointers trick), *always* check that an index is used on the correct list. So, the index cannot be used past the list (and therefore the element) lifetime. Note that we do not yet guarantee that we will never index an element deleted from the list, just that this region of memory will *always* belong to the list, i.e. be allocated.
1. No index will be used out of bounds. Easy, when crossing the end of a list, index becomes null. And we should assert it is not null at all points. Also, incrementing an invalid index could panic or yield the same invalid index.
1. Although it is not strictly required for memory safety, we can ensure to always yield valid elements. We *always* check that the element and index are in sync (e.g. using a timestamp - I chose a uuid named e_tag here to be slightly more resilient to attacks).

Finally, aliasing compliance:

* Breaking the pointer aliasing rules

Surprisingly, this is the easiest to reason. Since every access to the list goes through the `Index`/`IndexMut` traits, each reference held to an element also holds an immutable/mutable reference to the whole collection. So, the borrow checker ensures that we cannot have two mutable references on the same collection.

I am definite I have missed something, but my intuition says that it is not something that cannot be solved after a proper discussion.

Let’s see how the `Index` implementation could be. It is most probably wrong and I bet it segfaults. But the core idea is there. (If you use it in your production, just tell me to never apply for a job there :) ):

```rust
impl<T> Index<ListIndex<T>> for List<T>
{
    type Output = T;
    fn index(&self, index: ListIndex) -> &Self::Output
    {
        assert!(self.list_id == index.list_id,
            "Attempting to index the wrong List");
        assert!(!index.index.is_null(),
            "Attempting to use an invalid index");
        unsafe {
            assert!(index.e_tag == (*index.index).e_tag,
                "Attempting to use an invalidated index");
        }
        unsafe {
            &(*index.index).element
        }
    }
}
```

Let’s see the `ImmutableIndex` implementation:

```rust
impl<T> ImmutableIndex<ListIndex<T>> for List<T>
{
    fn begin(&self) -> ListIndex<T>
    {
        if self.first.is_null() {
            return ListIndex {
                list_id: self.list_id,
                e_tag: Uuid::new_v4(),
                index: std::ptr::null::() as *mut ListNode<T>
            }
        }

        let element = unsafe {
            &*self.first
        };

        ListIndex {
            list_id: self.list_id,
            e_tag: element.e_tag,
            index: self.first
        }
    }

    fn end(&self) -> ListIndex<T>
    {
        // ...
    }

    fn valid(&self, index: &ListIndex<T>) -> bool
    {
        self.list_id == index.list_id
        && !index.index.is_null()
        && unsafe {
            index.e_tag == (*index.index).e_tag
        }
    }
}
```

The `ForwardIndex` implementation will be similar (it checks for validity and updates pointer and e_tag).

```rust
impl<T> ForwardIndex<ListIndex<T>> for List<T>
{
    fn increment(&self, index: &mut ListIndex<T>)
    {
        assert!(self.list_id == index.list_id,
            "Attempting to index the wrong List");
        assert!(!index.index.is_null(),
            "Attempting to use an invalid index");
        unsafe {
            assert!(index.e_tag == (*index.index).e_tag,
                "Attempting to use an invalidated index");
        }

        index.index = unsafe {
            (*index.index).next
        };
        if !index.index.is_null() {
            index.e_tag = unsafe {
                (*index.index).e_tag
            };
        }
    }
}
```

The most interesting method is split/merge. Unfortunately, we have to be conservative here. When we merge a list into another, we have to invalidate *all* indices of the second list. If you can find a more clever invalidation scheme, I will be glad to hear it.

How do we invalidate all indices you may ask? Easier done than said. We change the list_id of the second list into a new random uuid!

Implementation is left to the reader :)

The final issue is the garbage collection of the reclaims vector. Unfortunately, the only answer I can think of is a `shrink_to_fit` implementation which invalidates all indices (using the new uuid trick). An alternative method could take a list of mutable indices as an argument and update the list_id of indices that can be kept valid. I don't think it is possible to reference count the indices, since it is not guaranteed that an index will always live longer than the collection, hence the Drop implementation has to (and cannot) check if the counter exists or not.

So, if I have not done a grave error somewhere, the only source of unsafety is uuid clash. It might happen, right? Well, there is a huge safeguard. Unless the uuid generator is predictable, there is a big difference between a code that works 99.9999% of the time and has a security issue 0.0001% of the time and a code that has a security issue 0.0001% of the time and *panics* 99.9999% of the time. And uuids have many more 9s. After all, if someone does not notice trillions of panics, maybe he deserves to be hacked... :)

That's all folks :)

---
---

### References

1. [Ruby Sinatra][sinatra]
1. [std::collections::LinkedList][std::LinkedList]
1. [ixlist, an implementation of Linked Lists][ixlist]
1. [linked-list crate][linked-list crate]
1. [A random reddit thread][reddit thread]
1. [Rustonomicon - what unsafe can do][Rustonomicon]
1. [Source code to my implementation][source code]
1. [Slides of my presentation][slides]
1. [Learning Rust With Entirely Too Many Linked Lists][Linked List Book]
1. [Rust Book on Traits (Orphan Rule)][Rustbook Traits]
1. [Coq proof assistant][Coq]

[sinatra]: http://sinatrarb.com
[std::LinkedList]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[ixlist]: https://bluss.github.io/ixlist/target/doc/ixlist/struct.List.html
[linked-list crate]: https://crates.io/crates/linked-list
[reddit thread]: https://www.reddit.com/r/rust/comments/7zsy72/writing_a_doubly_linked_list_in_rust_is_easy/
[Rustonomicon]: https://doc.rust-lang.org/nomicon/what-unsafe-does.html
[source code]: https://gitlab.com/softsilverwind/linked-list-test
[Points]: /assets/01/points.png
[slides]: /assets/01/linked-list.pdf
[RIIR]: https://github.com/ansuz/RIIR
[Linked List Book]: https://cglab.ca/~abeinges/blah/too-many-lists/book/
[Rustbook Traits]: https://doc.rust-lang.org/book/ch10-02-traits.html
[Coq]: https://coq.inria.fr/
