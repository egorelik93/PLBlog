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
it cannot be moveable. Thus, unlike Rust, we do need true immoveable types - in keeping with the Rust style, we'll say that `Move` is a trait like `Sized`, and that immoveable types impl `!Move`.

Multiple lifetimes can be combined into a narrower lifetime using the `*` operator.

```
struct Example2 {
  source1: Pinned<i32>,
  source2: Pinned<i32>,
  ptr: &'source * 'source2 i32
}
```

Unlike Rust, NewLang does not have lifetime inference. Instead, functions preserve the lifetime of their inputs. We can still write

```
fn id(ptr: &i32) -> &i32 {
  ptr
}
```

This function can be invoked on a reference that has any sort of lifetime, and the result will have the same lifetime. In this way, functions *preserve* order.

If a function has multiple arguments, the lifetime of the result is the intersection of all the argument's lifetimes.

```
fn f(a1: &i32, a2: &i32) -> &i32 {
  a1
}
```

This is very different from Rust, which understands that only `a1`'s lifetime mattered. Later we will come back to ways to get the Rust behavior.

Note that `&i32` is not just `&'l i32` with some sort of inferred lifetime parameter either. Think of the lifetime not as part of the reference type but as an externally
applied constraint, which functions must preserve. If you need to explicitly apply this constraint, such as in a struct definition, the preferred syntax is `'l % i32` (the use of the `%` symbol specifically is tentative).

```
struct Example {
  source: Pinned<i32>,
  ptr: 'source % &mut i32
}
```

This operator can be applied to any type. If we took just what has been described so far, it would seem like we could somehow have a type like `'l % i32`. We can indeed write, and it would seem like writing a function like
the following would constrain the result in this way:

```
fn copy(ptr: &i32) -> i32 {
 *ptr
}
```

The reason why `i32` isn't actually order-constrained is that we know that `i32` outlives all lifetimes. In Rust, we write this fact as `i32: 'static`. We'll use the same "outlives" syntax in NewLang,
and in fact because of order preservation it becomes much more important.

For every non-dyn type in Rust, there is some lifetime that we know it outlives. In Rust, that is just an automatic consequence of how its lifetimes work - a type can only be constrained by a lifetime
that is passed to it as an input. That is unlike NewLang, where lifetime constraints are applied externally. However, we want all such existing types to behave the same way in NewLang. That means
a struct in NewLang should automatically derive information about what lifetimes it outlives. For such types, the lifetime application operator has limited effect; the type can only be constrained by those
variables that are in the lifetime it outlives.

The exception from Rust is borrowed pointer types. For the earlier lifetime application operator to work, `&T` and `&mut T` must not outlive any potential lifetimes. Generally, this means that structs
that in Rust take lifetime parameters for inner pointers will, if unchanged, have no knowledge of outliving anything in NewLang.

```
struct BadExample3<'l, 'm> {
  ptr1: 'l % &i32,
  ptr2: 'm % &i32
}
```

Thus, in NewLang, this needs to be structured a bit differently. Rather than specifying order constraints, the job of a lifetime parameter to a struct is to instead specify "outlives". The job of actually
specifying the order constraint should then be applied externally. To specify that a pointer "outlives" a lifetime, we use the same syntax as Rust does for dyn-objects.

```
struct Example3<'l, 'm> {
  ptr1: &i32 + 'l,
  ptr2: &i32 + 'm
}
```
