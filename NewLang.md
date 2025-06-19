These are design notes for a programming language in a similar space to Rust, but based on some different foundations.

For ease of explanation, I will stick to Rust syntax as much as possible.

I want to start by explaining the 'lifetime system', which is very different from Rust's. It is ultimately based on the idea of variables having an "order" amongst themselves.
This is most clear inside of structs:

```
struct Example {
  source: Pinned<i32>,
  ptr: &'source mut i32
}
```

This example is what in Rust folklore is called a self-referential struct. It contains a value, and then a pointer to that value. A "lifetime" in "NewLang" is a set of variables or fields. A variable or type
that contains this lifetime has its order "constrained" - it must be consumed before any of the variables in that set are. Struct fields are one of a few places in NewLang where one can explicitly name which fields
are part of this set. If one has a variable `x`, you cannot just name its lifetime as `'x`. This is because lifetimes are tied to "places", not values - `x = y` cannot imply that `'x = 'y`.
However, variables do still remember the notion of order - we can refer to both `e.source` and `e.ptr`, and NewLang will undersand that `e.ptr` must be consumed first.

`Pinned` here should not be confused with Rust's `Pin`, although they are a related idea. In NewLang, one can take the lifetime of an i32 field, and it won't actually do anything.

```
struct BadExample {
  source: i32,
  ptr: &'source mut i32
}
```

You can write this type, but not actually create any instances of it. An i32 that is being referenced this way is not actually "borrowed", it can be used in any order.
`Pinned` is what actually enforces the order constraint. In NewLang, when a variable `x: T` is borrowed, `x` implicitly becomes of type `Pinned<T>` for the duration of the borrow.
While it is borrowed, a variable of type `Pinned<T>` cannot be used. When the borrow ends, however, it automatically goes back to just being of type `T`.

Structs that contain pointers to themselves cannot be moved without breaking the pointer (or some kind of update mechanism), so for this to be possible, `Pinned<T>` and any struct containing
it cannot be moveable. Thus, unlike Rust, we do need true immoveable types - in keeping with the Rust style, we'll say that `Move` is a trait like `Sized, and that immoveable types impl `!Move`.

Multiple lifetimes can be combined into a narrower lifetime using the `*` operator.

```
struct Example2 {
  source1: Pinned<i32>,
  source2: Pinned<i32>,
  ptr: &'source * 'source2 i32
}
```
