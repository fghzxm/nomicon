# Destructors

What the language *does* provide is full-blown automatic destructors through the
`Drop` trait, which provides the following method:

<!-- ignore: function header -->
```rust,ignore
fn drop(&mut self);
```

This method gives the type time to somehow finish what it was doing.

**After `drop` is run, Rust will recursively try to drop all of the fields
of `self`.**

This is a convenience feature so that you don't have to write "destructor
boilerplate" to drop children. If a struct or enum has no special logic for
being dropped other than dropping its children, then it means `Drop` doesn't
need to be implemented at all!

Note that the recursive drop behavior applies to all structs and enums
regardless of whether they implement Drop. Therefore something like

```rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

will have its data1 and data2's fields destructors whenever it "would" be
dropped, even though it itself doesn't implement Drop. We say that such a type
*needs Drop*, even though it is not itself Drop.

Similarly,

```rust
enum Link {
    Next(Box<Link>),
    None,
}
```

will have its inner Box field dropped if and only if an instance stores the
Next variant. (Note that the standard library's `LinkedList` uses a custom
`Drop` implementation because automatic recursive dropping is prone to
overflowing the stack, but that's a separate issue.)

In general this works really nicely because you don't need to worry about
adding/removing drops when you refactor your data layout. Still there's
certainly many valid use cases for needing to do trickier things with
destructors.

For instance, a custom implementation of `Box` might write `Drop` like this:

```rust
#![feature(ptr_internals, allocator_api)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::mem;
use std::ptr::{drop_in_place, NonNull, Unique};

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>())
        }
    }
}
# fn main() {}
```

and this works fine because when Rust goes to drop the `ptr` field it just sees
a [Unique] that has no actual `Drop` implementation. Similarly nothing can
use-after-free the `ptr` because when drop exits, it becomes inaccessible.

However this (for the sake of demonstration) wouldn't work:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Box<T> }

impl<T> Drop for SuperBox<T> {
    /// hYpEr-OpTiMiZeD! Deallocate the box's memory
    /// without `drop`ping the contents!!!
    /// (Note that leaking objects does not inherently constitute
    ///  undefined behavior in Rust)
    fn drop(&mut self) {
        unsafe {
            let c: NonNull<T> = self.my_box.ptr.into();
            Global.deallocate(c.cast::<u8>(), Layout::new::<T>());
        }
        // `my_box` dropped after this method returns
    }
}
# fn main() {}
```

After we deallocate the `box`'s ptr in SuperBox's destructor, Rust will
happily proceed to tell the box to Drop itself and everything will blow up with
use-after-frees and double-frees.

Note that `Drop::drop` taking `&mut self` means that even if you could suppress
recursive Drop, Rust will prevent you from e.g. moving fields out of self. For
most types, this is totally fine.

The classic safe solution to overriding recursive drop and allowing moving out
of Self during `drop` is to use an Option:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, GlobalAlloc, Global, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    /// hYpEr-OpTiMiZeD! Deallocate the box's memory
    /// without `drop`ping the contents!!!
    fn drop(&mut self) {
        // Need to set the `box` field to `None`
        // to prevent Rust from trying to `drop` it.
        let my_box = self.my_box.take().unwrap();
        unsafe {
            let c: NonNull<T> = my_box.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
            // Prevent double free.
            mem::forget(my_box);
        }
    }
}
# fn main() {}
```

However this has fairly odd semantics: you're saying that a field that *should*
always be Some *may* be None, just because that happens in the destructor. Of
course from a safety perspective this makes every sense: you can call arbitrary
methods on `self` during the destructor, which no longer contains a live `Box`
after you've taken it out, and the `Option` is there so that one does not simply
access that nonexistent `Box` further without matching on the `Option` (and fail
epically and deterministically).

But besides that, this solution also have some drawbacks that can be pretty
serious in the eyes of someone who for some reason is writing Unsafe Rust in the
first place:
1. Every run-of-the-mill method on `SuperBox` is now presented with an
   `Option<Box<T>>` that you know can't be `None`, but Rust doesn't know that
   and will force you to `unwrap` it every time you want to do anything with the
   inner `Box`, nor can the optimizer prove that and remove the check unless
   the entire call chain between the `unwrap` and the construction of the
   `SuperBox` *and* every function call in between that takes a mutable reference
   to it gets inlined; and (of course you can use `unwrap_unchecked` and risk
   Undefined Behavior when you inadvertently implement a method/trait that really
   sets `my_box` to `None` outside of the destructor, but even then...)
2. There's a hidden cost associated with the use of `Option`: Whereas `Box` is
   eligible for Guaranteed Niche Optimization, `SuperBox` is not because it
   itself wraps an `Option<Box>` that already occupied that niche. In other words,
   it's guaranteed that `size_of::<Option<Box>>() == size_of::<Box>()`, but it's
   likely the case that `size_of::<Option<SuperBox>>() > size_of::<SuperBox>()`.

With all of that being said, in general `Option` is the only option that safe Rust
code wishing to implement this kind of behavior has access to. For you, the
intrepid Unsafe programmer though, if the above issues make `Option` a non-starter
for you, use `std::mem::ManuallyDrop`:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, GlobalAlloc, Global, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem::{self, ManuallyDrop};

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> {
    // Dropping a `ManuallyDrop<T>` doesn't drop the inner `T`.
    my_box: ManuallyDrop<Box<T>>,
}

impl<T> Drop for SuperBox<T> {
    /// hYpEr-OpTiMiZeD! Deallocate the box's memory
    /// without `drop`ping the contents!!!
    fn drop(&mut self) {
        unsafe {
            let c: NonNull<T> = self.my_box.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
        // `my_box` is recursively dropped,
        // but `<Box<T> as Drop>::drop` doesn't run.
        // Double free no more!
    }
}

impl<T> DerefMut for SuperBox<T> {
    type Target = T;
    fn deref_mut(&mut self) -> &mut T {
        // Accessing the inner value of `ManuallyDrop<T>` is safe.
        // In fact, `ManuallyDrop<T>` implements
        // `Deref<Target = T>` and `DerefMut<Target = T>`.
        let ptr = self.my_box.ptr;
        unsafe { ptr.as_mut() }
    }
}
# fn main() {}
```

`ManuallyDrop` have other uses as well, such as [controlling drop order][dropck]
without implicitly depending on Rust's default drop order.

[dropck]: dropck.md
[Unique]: phantom-data.md
