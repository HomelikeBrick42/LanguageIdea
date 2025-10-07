# Language Idea

This language is mostly similar to rust (currently) but with alot of changes

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

TODO: should non-readable references exist?

TODO: should there be another out-parameter reference which cant be dropped, only written to? It could be useful for constructing "pinned" objects in-place

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
- `Vec::drain` could return a type just containing a `&own [T]` member and some info to move the elements back when its dropped, it wouldnt even have to drop the `T`s manually because it holds a `&own [T]`, and the destructor wouldnt interfere with using the `&own [T]` either because it partially-borrows everything but that member

## "Interrior mutability" types

This isnt really interrior mutability, but its similar enough to rust that I'll call it that

- A version of rusts `Cell` will exist to be able to mutate the state of an `enum` when you only have a shared mutable reference (because you need a unique mutable reference to reassign it)
  <br>
  This will be at the cost of not being able to reference anything inside the `Cell`, just like rust, because it could cause reference invalidation and then use-after-frees

- A version of `RefCell` will also exist to be able to get a unique reference from a shared mutable reference, at the cost of runtime checks to make sure multiple unique references are not created

NOTE: `UnsafeCell` is not needed as a special magic language-builtin anymore, since the uniqueness can just be tracked by the `Cell`/`RefCell` and the compiler already expects mutability when doing aliasing optimisations on `Cell`/`RefCell` because you need a mutable shared reference to get a shared reference

## Partial Borrowing

TODO

## Destructors

TODO: current idea is to have `Drop::drop` take a partially-borrowed `&own` reference, any members not specified to be partially-borrowed by the method signature will be dropped by the compiler before `Drop::drop` is run
<br>
Also this allows you to avoid the hack where `ManuallyDrop` is put on members so they could be moved out of in the destructor, instead the `&own` references allows you to move out of members
<br>
In fact, `ManuallyDrop` could be implemented entirely within user code just by leaking an `&own` reference to its only member therefore causing its drop glue to never be run
<br>
(NOTE: this also could fix weird "dropck eyepatch" stuff in current rust)

## Const arguments

Parameters can be annotated with `const` to make them require a compile-time value, const generics do not exist

## Implicit arguments

Every function will have implicit arguments in `[]`, and then explicit arguments after in `()`

If explicit arguments arent specified, the compiler will try to infer what it should be from context, and if that fails, it will then attempt to pick some default value for it

TODO: figure out the syntax for making something the default value in a scope

TODO: figure out the syntax for making all functions in a scope take a certain implicit argument

## First class types

There will be no "generics", only `const` arguments with the `Type` type, and then those const arguments can be used as the type of variables, other parameters, etc

## Const

There will be no `const fn` in this language, unlike rust, everything will be able to be run at compile time

`main` will take a special `IO` value as a runtime parameter (which will have zero size), and then things like FFI, printing stuff to the console, etc, will require this `IO` value

The `IO` value will not be able to be constructed manually, so the only way to get one is though `main`'s argument, and so you cannot obtain one at compile time

This avoids all the coloring issues that rust has with `const fn`s and `const trait`s

As an example, if there is some `Option::map` method, it doesnt need to care whether the function its passed does side effects, it just wants to call the function
<br>
If the caller wants to cause side effects, they can just pass a closure that captures the special `IO` value, and `Option::map` never needs to know

## Extra `IO` ideas

There could be "restrictions" on `IO`, like a special struct for printing that just stores `IO`, and then you can pass that more restricted type to functions that want to print stuff, without giving them full access to do other side effects

There could be other structs too for only writing to a specific directory, etc, which you could pass to mods/plugins that applications

## Traits

There are no traits. "Trait bounds" will be replaced by just taking a struct with some functions in it as an (implicit/explicit) argument

Trait implementations will just be a value of the custom vtable struct, which means the user can also override any trait implementation that they want by just passing a diffferent vtable value when calling a function

Implicit arguments and custom default values will be useful for automatically passing the correct vtable to methods
