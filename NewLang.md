These are design notes for a programming language in a similar space to Rust, but based on some different foundations. This is a work in progress.

For ease of explanation, I will stick to Rust syntax as much as possible.

## Lifetimes

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

NewLang now understands that this struct outlives `'l * 'm`. No matter what lifetime constraint is applied to `Example3`, only those that share variabled with `'l` or `'m` will have any effect. Furthermore,
they will distribute over the fields. Thus, if we have `e: 'n % Example3<'l, 'm>`, we can extract `e.ptr1: ('l * 'm) % (&i32 + 'l)` and `e.ptr2: ('l * 'm) % (&i32 + 'm)`. Their respective outlives specifications
mean that these become `e.ptr1: 'l % &i32 + 'l` and `e.ptr2: 'm % &i32 + 'm`.

As a note, I'm actually undecided whether `Example3<'l, 'm>` is already constrained to `'l * 'm` or not. You cannot name this type once `'l` and `'m` are gone, and generally I want to say that if the type
cannot be named, the value's lifetime must have ended. On the other hand, the individual fields could live longer. I'll need to come back to this.

Outlives specifiers can be dropped at any time, however. This means a type can specify that it outlives a lifetime that will shortly cease to exist. This is relevant to function return types. In NewLang,
function parameters are one of the few places where we allow directly naming the lifetime of a variable, and the function return type is allowed to say that it outlives one or more of those lifetimes. Coming back to an earlier example:

```
fn f2(a1: &i32, a2: &i32) -> &i32 + 'a1 {
  a1
}
```

This is how we say, like in Rust, that only the first lifetime mattered in the result. While the result cannot directly name `'a1` in its type since `a1` is no longer in scope, this pattern allows
us to say that the lifetime of `a2` can be ignored.

For the same reason as my earlier note, it is not quite clear to me what it would mean to pass `'a1'` here as a lifetime parameter to a struct being returned, such as `Example3`. Maybe it would make sense
here to use explicit lifetime parameters on the function, since we are already doing so on the type. However, I have not figured out all the interactions of lifetime generics with order preservation.
For now, assume that all lifetime parameters on a function must be explicitly applied as constraints to the return type.

However, the only reason we need lifetime parameters in `Example3` is because we have pointers of two different lifetimes inside. If we only had one lifetime, it would be much nicer to just write

```
struct Example4 {
  ptr: &i32
}
```

To make this work, NewLang allows outlives specifiers to be applied to *any* type.

```
fn f3(a1: &i32, a2: &i32) -> Example4 + 'a1 {
  Example4 { ptr: a1 }
}
```

Even if no outlives specifier is specified on a return type,
functions cannot return anything of shorter lifetime than its input, 
so NewLang understands the return at least outlives the intersection of the lifetimes of the function's parameters.
Thus, in the above example, one never needs to specify `Example4 + 'a1 * 'a2`. 
This is not to be confused with the lifetimes of the input values themselves. 
If all the input values themselves are known to outlive some lifetime,
then NewLang can prove that the result outlives *that* lifetime.

## Trait Objects and Closures

So far, this system probably doesn't seem any more useful than Rust's, possibly with a more convenient default. As I mentioned, every struct in Rust should behave the same way in NewLang. Where NewLang differs drastically is in *trait objects*.

First, note that NewLang does not use the `dyn` keyword, much like earlier versions of Rust.

Say we have the following trait and impl:

```
trait MyTrait {
  fn a(self) -> A { ... }
  fn b(self) -> B { ... }
}

struct MyStruct { ... }

impl MyTrait for MyStruct { ... }
```

In NewLang, it is possible to write this:

```
fn new() -> MyTrait {
  MyStruct { ... }
}
```

In NewLang, traits are types. These types behave differently from Rusts structs however. If you use the type immediately, there are no issues.

```
let m = new();
m.a()
```

You can pass them to a function.

```
fn get_a(m: MyTrait) -> A {
  m.a()
}

let m = new();
get_a(m)
```

You can even include them in a struct.

```
struct Container {
  m: MyTrait
}
```

In NewLang, trait objects participate in the same preservation of order that all types do. If we instead have a struct like

```
struct MyRefStruct {
  ptr_a: &A,
  ptr_b: &B
}

impl MyTrait for MyRefStruct {
  fn a(self) -> A {
    *self.ptr_a
  }

  fn b(self) -> B {
    *self.ptr_b
  }
}
```

`MyRefStruct` obviously will have a lifetime. What happens when creating the trait object is what happens with all function; the lifetime will get transferred to `MyTrait`.

```
fn from(m : MyRefStruct) -> MyTrait {
  m
}
```

What primarily distinguishes trait objects in NewLang from other types, and types in Rust, is that there is no known lifetime that they outlive. As in Rust, you sometimes need to explicitly 
specify that the trait object outlives a particular lifetime. Unlike in Rust, we do not and cannot infer one if it is not specified.

This extends to closures, allowing us to write functions like:

```
fn example5(a: &i32) -> Fn(&i32) -> i32 {
  |b| *a + *b
}
```

We can also write

```
fn example6(a: &A) -> Fn(&B) -> &A {
  |b| a
}
```

The final lifetime is the intersection of both arguments. The lifetime of `&A` gets transferred to the closure. When calling a closure or any other trait object,
since the object is actually one of the inputs, its lifetime also gets transferred to the final result along with any arguments.

As a convenience, when writing a curried function in NewLang, we can often drop `Fn` and just write 

```
fn example7(a: &A) -> &B -> &A {
  |b| a
}
```

Whether the closure object in this case implements `Fn`, `FnMut`, or just `FnOnce` can be determined from the environment it takes in.

The transfer of lifetimes allows for an "unusual" implementation of `Move` for trait objects. Instead of literally moving a dynamically-sized
value, we often can silently take a pointer to the original contents and implictly construct a new, small trait object that references it. Ownership is transferred
without actually moving anything. This allows us to implement a function like the following without dynamically-sized stack objects:

```
fn extract_trait(m: MyContainer) -> MyTrait {
  m.m
}
```

Being trait objects, this works for closures too.

```
fn invoke(f: FnOnce(A) -> B, a : A) -> B {
  f(a)
}
```

In practice, a decent optimizer should usually be able to monomorphize this for particular closure-implementing types where appropriate.

As in Rust, sometimes we do need to know that a trait object outlives some lifetime. Rust already allows us to specify this on traits, and as mentioned earlier, NewLang allows
this to be specified on any type. In some cases it can prove that a particular returned trait object, even if it does not explicitly specify it, must outlive a particular lifetime because of its inputs.
In such cases, it is legal to then add on the lifetime specifier to an object that did not previously have it.

## Linear Types and Borrowed References

Before moving on to the meat of the next section, we need to get some prerequisites out of the way. Unlike Rust, NewLang is genuinely *linear-typed*. The `Drop` trait is used for all types that can be implicitly
dropped, not just those that explicitly need a custom implementation to drop resources. Most types are still expected to implement `Drop`, but whether this is auto-derived like Rust's `Sync` or implemented by default like
Rust's `Sized`, I have not quite figured out yet. I won't address the exact implementation of `Drop` quite yet, but I will note that NewLang does not have any of Rust's magic for `Drop`; implementors will have to manually drop
their internals.

On the other hand, we have to deal with the bane of Linear type systems - exceptions/panics. The way NewLang avoids this issue is to require even true linear types to implement a *separate* Drop-like trait just for cleanup during
a panic, which we are tentatively calling `Unwind`. This trait is a little more like Rust's `Drop`; every type is assumed to have some sort of `Unwind` implementation, even if that is to abort the program. In practice, a good linear type should be designed to allow for a safe non-aborting `Unwind` instance.

With that out of the way, we can move on to the main subject of this section, which are the various borrowed references available in NewLang. As previously discussed, borrowing a value in NewLang both creates a reference and places the original value under a `Pinned` type. NewLang does have the basic `&` and `&mut` references, but it also has a number of other borrowed reference types. The core such type is (tentatively) called `&tmut`, meaning "typed mut", which allows
*changing* the type under `Pinned`.

```
let tmut x = 5;
let y: &tmut i32/bool = &tmut x;
*y = true;
x == true
```

A reference of type `&tmut A/B` is a pointer that currently points to a value of `A`, but which carries an obligation to have a value of type `B` written to it before being dropped. Unlike `&mut A`, `&tmut A/B` is a true linear type and cannot be implicitly dropped, unless it is `&tmut A/A`. When we do write a value of type `B` to the reference, it changes the type of the reference to `&tmut B/B`. We can actually write any `Sized` type to this location
as long as it we statically know it fits in the space available. When we create an `&tmut A/B` by borrowing a value of `A`, that value then becomes of type `Pinned<B>`. We then get access to a value of type `B` once the borrow ends.

The other borrowed reference types are just synonyms for specific cases of this. `&in A` is `&tmut A/()`, and `&out A` is `&tmut ()/A`.

We can now define NewLang's version of `Drop`:

```
trait Drop {
  fn drop(&in self);
}
```

`&in` is NewLang's terminology for what the Rust community calls a "move" pointer. It is semantically the same as the pointed type when the latter is `Sized`, but it allows
us to work with `!Sized` types.

We can also abbreviate `&tmut A/A` as `&tmut A`. This type is for the most part the same as `&mut A`, but it carries a key difference. That key difference means that we cannot turn an `&mut A` into an `&tmut A` and temporarily
write a different type into it. The issue is with how `&tmut` interacts with unwinding or any sort of diverging effect. With `&mut`, because the type never changes there is no question of what type needs to be unwound. We can just drop the `&mut` and then unwinding `Pinned<A>` as `A` is type-safe (whether that is logically safe is something only the user can answer). With `&tmut` though, we need to be very careful; we cannot just drop the reference. Furthermore, because the type
can change, we have no way of actually knowing what type is present when we try to unwind the associated `Pinned<A>`. Only the reference knows what the current type is, so we have no choice but to give it responsibility for unwinding its current value. However, that creates another problem - if a panic occurs after an `tmut A/A` is dropped but before the borrow ends, possibly because it was dropped in another function, there is no reference available for us to unwind, only the `Pinned` value. Somehow, NewLang has to know whether or not the value inside `Pinned` was already unwound, and act accordingly. This is not something can be determined statically. For this reason, `&tmut A/B` cannot just be a simple pointer; it needs to also carry some information about how to let the runtime know that the pointed value is being unwound.

Strictly speaking, `&tmut A` does not actually require this in order to be safe - in fact, for the longest time there was no separate `&tmut`; I was just using `&mut A/B`, and more recently distinguishing between `&mut A/A` and `&mut A`. However, the change in size and capabilities is too confusing to not have some clearer means of distinguishing them. Any `&tmut A/B` can itself be borrowed as `&mut A`, so this is my current answer. Methods that mutate a borrowed value but are not overly concerned about panics can probably continue to use `&mut`.

Because unwinding information needs to be stored somewhere, when borrowing a value as `&tmut`, instead of the original value becoming `Pinned<B>`, the type we use is called `Susp<B>` (WIP Note: Haven't finalized if we're using the `Susp` name for this purpose. Alternative is `TPinned`). Like `Pinned<B>`, `Susp<B>` mostly preserves the checks that Rust does for borrowed values. However, it does have some tricks.

```
let tmut x = 5;
let y: &tmut i32/bool = &tmut x;
let z = x.defer && true;
*y = true;
z == true
```

`.defer` is a special syntax in NewLang. It is a bit like Rust's `.await`, but its effect is scoped to the current line. When `x` is a `Susp<B>`, within the expression `x.defer` will be an expression of type `B`.
The result of the expression will then itself become `Susp`. Past that line, `x` is then out of scope; the lifetime and ownership of the final value are then taken over by `z`.

One might assume that `.defer` is just reordering lines until when the borrow ends. Actually, that is not what it does. As with unwinds, there is a difference between deferring to when the borrow ends versus to when the reference
gets dropped, and we need to consider how unwinding will account for these deferred lines, which can further change the type beyond what the reference knows about. What seems to be necessary is to execute these deferred lines exactly when the reference gets dropped. This requires runtime support, but mostly the same as what we needed already to ensure that unwinding is safe - this is why I have been intentionally vague about how that check is performed. This unwind check can actually be regarded as a special case of a deferred line that is built-in to every borrow.

(WIP: `TPinned`, or an alternate name, would stick to only the unwind check rather than allowing a callback. Not clear if this is worthwhile to have separately).

There is still some heavy language magic going on in order to enable `.defer` to be used with `Susp`, and that results in a hard restriction - `Susp` can only use `defer` in the block where the borrow begins in the first place. This because in reality, `Susp` is taking in a single closure of all the deferred logic when the borrow occurs. We could manually write this out like this:

```
let tmut x = 5;
let (x, y: &tmut i32) = Susp::borrow_tmut(x, |z| z + 1);
*y *= 2;
x // == 11
```

The tuple being returned by the borrow is actually self-referential, so that `y` is order constrained to be before `x`.

Most significantly, this limitation on `Susp` means that `.defer` *cannot* be used with a `Susp` that came from a self-referential struct. 

Mathematics suggests that ideally, there are some laws we would want a `Susp`-like type to obey. `Susp` itself does not obey these, so I am tentatively calling this idealized type `Defer`. One law essentially amounts to lifting the limitations on `.defer` - more explicitly it means that this type has a `map` operation
that can be used even when it is being borrowed. It is actually possible to implement this, but it does not come for free; one can use a linked list of callbacks whose nodes are on the stack. Writing and dropping a reference then requires traversing this list, which is a bit expensive for what should just be a "write" operation. The other law is much more problematic - we can express it as the existence of a function `fn((Defer<()>, '0 % A)) -> A`. This says that once a value has been deferred to a `()`, we can actually hide it, even while it is still borrowed. This amounts to being able to ignore a lifetime under certain conditions. Implementing such a function under normal calling conventions is impossible. If we were to try to implement `Defer`, my current idea is to add yet another reference type, called `DMut<A, B>` - just as `Susp` is produced alongside `&tmut A/B`, `Defer` would be produced alongside `DMut<A, B>`. `DMut<A, B>` would be treated specially by NewLang - a function that returns a `DMut<A, B>` or a type containing it would actually be implemented using continuation-passing style rather than the normal stack - this should allow its order constraint to be forgotten if conditions are appropriate. Otherwise, `DMut<A, B>` is essentially the same as `&tmut A/B` and can itself be borrowed as the latter. (WIP: it may make sense to give `DMut<A, B>` more flexible unwinding options that `&tmut A/B`, in order to make it safe to catch.)

While mathematically we would want all order constraints to use `Defer`, in practice I believe the much cheaper `Susp` will cover most uses. The point of having `Defer` as well is to establish what the most general semantics of borrowing are. `Susp` and `Pinned` can be seen as optimized special cases of `Defer`.

It may be possible to provide an extremely limited form of `.defer` to be used with `Pinned`. This seems to make sense when using a struct or enum constructor, and it may even be sound for non-side-effectful functions, like const functions. I am not sure if it is worthwhile to allow this, however.

## Advanced .defer tricks

Whether `Susp` or `Defer`, we can use deferred callbacks to set up some tricks. For example, we can borrow a `Box<A>` as an `&in A` and leave a `Susp<()>`. Internally, this works by setting up the deferred callback
to deallocate the box. Another example is, if we have some sort of channel, we could set up a reference whose callback automatically pushes to that channel. Many "Guard" types that combine a pointer with some sort of release
mechanism now become optional.

There is also another alternative to `Susp` or `Defer`. Any type on the "deferred" side of an order relationship can be places into an `FnOnce` closure.

```
let tmut x = 5;
let y: &tmut i32 = &tmut x;
let z = || x.defer + 10;
*y *= 2;
z()
```

This works because, just as closures and trait objects have indefinite lifetimes due to being able to contain references, they also cannot escape being on the deferred side of an order constaint due to the possibility of containing a `Susp` or similar type. They then cannot be invoked until free of the order constraint - by which point the deferred value is guaranteed to be available.

More generally, any struct or enum can contain `Susp`-like types, even `Pinned`. To allow this to work, it is possible to pass order-constrained `Susp`-like values to a constructor. We do not use `.defer` syntax when doing this explicitly. However, there are restrictions; especially with `Pinned` and `Susp`, we cannot even fake moving these values, so this can only be done in the block where they initially get borrowed. NewLang will set up the 
stack allocation so that the `Pinned`/`Susp` value gets pinned in place, and nothing needs to get moved. Ownership of lifetimes will still get transferred.

```
struct MySuspContainer { susp: Susp<i32> }

fn f() -> i32 {
  let tmut x = 5;
  let y = &tmut x;
  let z = MySuspContainer { susp: x };
  *y *= 2;
  z.susp
}
```

These "deferred" closures allow us to pull off a cool trick:

```
fn to_parts<A, B>(f: FnOnce(A) -> B) -> (Fn() -> B, '0 % &out A) {
  let a : A;
  let outA = &out a;
  let g = || f(a.defer);
  (g, outA)
}

fn from_parts<A, B>(f : Fn() -> B, outA : 'f % &out A) -> A -> B {
  |a| {
    *outA = a;
    f()
  }
}
```
