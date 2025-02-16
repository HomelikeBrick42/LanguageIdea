# Language Idea

This language is mostly similar to rust (currently) but with a few changes

## Reference types

This language is trying to be less restrictive than rust, so there are 12 reference types instead of just 2

### Reference categories

Every reference type has one "capability" from these three categories

- Readability
  - Readable
  - Non-readable
- Writability
  - Writable
  - Non-writable
- Access
  - Shared
  - Unique
  - Owned

NOTE: "shared" and "unique" are whether other writable references can exist pointing to that same valaue
<br>
(maybe "noalias" would be a better name for "unique"? though that is equally as unclear and only slightly better because of LLVM connotations and being a more esoteric name so that new people dont assume that "unique" means unique to other immutable references aswell)

### Syntax

The syntax for references has not yet been decided

### Comparisons to rust

- A `&T` from rust which does not contain an `UnsafeCell` is readable, non-writable, and unique
- A `&T` from rust which does contain an `UnsafeCell` is readable, writable, and shared
- A `&mut T` from rust is readable, writable, and unique

### Why shared mutability?

Rust disallows shared mutability to prevent these kinds of cases

```rs
let mut x: Option<String> = Some(String::new("hello"));

let y: &String = x.as_ref().unwrap();

x = None; // this destroys the `String` that `y` is referencing by changing the variant

println!("{y}"); // causes a use-after-free, this is UB!!
```

```rs
let mut v: Vec<i32> = Vec::with_capacity(1);
v.push(1);

let elem: &i32 = &v[0];

v.push(2); // this makes `v` reallocate its internal buffer, invalidating `elem`

println!("{elem}"); // again a use-after-free, this is UB!!!
```

But disallowing shared mutability entirely just for these few cases is overly restrictive, so instead this language allows shared mutability but then will only require "unqiue" mutable access to values in these specific cases like reassigning enums and pushing to `Vec`s in a way that could reallocate
<br>
(There are ways around having unique access, see the "interrior mutability" section below)

### Why should references be able to have ownership?

- It allows stuff like rusts `dyn FnOnce()` to exist and not be magic, since you can just pass `&own dyn FnOnce()` to the `call_once` method
- It allows moving out of slices (like with `IntoIterator` or possibly indexing/pattern matching?)
- It allows bump allocators to return `&own` references to values that are allocated, making the value's `Drop` impl get called when the `&own` goes out of scope and making sure the reference lifetime is tied to the allocator

## "Interrior mutability" types

This isnt really interrior mutability, but its similar enough to rust that I'll call it that

- A version of rusts `Cell` will exist to be able to mutate the state of an `enum` when you only have a shared mutable reference (because you need a unique mutable reference to reassign it)
  <br>
  This will be at the cost of not being able to reference anything inside the `Cell`, just like rust, because it could cause reference invalidation and then use-after-frees

- A version of `RefCell` will also exist to be able to get a unique reference from a shared mutable reference, at the cost of runtime checks to make sure multiple unique references are not created

NOTE: `UnsafeCell` is not needed as a special magic language-builtin anymore, since the uniqueness can just be known by the `Cell`/`RefCell` and the compiler already expects mutability when doing aliasing optimisations on `Cell`/`RefCell` because you need a mutable shared reference to get a shared reference

## Partial Borrowing

TODO

## Destructors

TODO: current idea is to have `Drop::drop` take a partially-borrowed &own reference, any members not specified to be partially-borrowed by the method signature will be dropped by the compiler
<br>
Also this allows you to avoid the hack where `ManuallyDrop` is put of members so they could be moved out of in the destructor, instead `&own` references to partially-borrowed members can just be leaked
<br>
(NOTE: this also could fix weird "dropck eyepatch" stuff in current rust)
