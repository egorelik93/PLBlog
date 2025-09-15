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

If `T : 'l`, then it also the case that `('l % T) : 'static`. Whether the reverse is true is not yet clear. 

## Trait Objects and Closures

So far, this system probably doesn't seem any more useful than Rust's, possibly with a more convenient default. As I mentioned, every struct in Rust should behave the same way in NewLang. Where NewLang differs drastically is in *trait objects*.

First, note that in NewLang, one ordinarily does not want to use the `dyn` keyword with trait objects.

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

Here is what is actually happening. The transfer of lifetimes allows for an "unusual" implementation of `Move` for trait objects. Instead of literally moving a dynamically-sized
value, we often can silently take a pointer to the original contents and implictly construct a new, small trait object that references it. Ownership is transferred
without actually moving anything. This allows us to pass a trait object as an argument to a function like the following without dynamically-sized stack objects:

```
fn invoke_trait() -> A {
  let m = new();
  get_a(m)
}
```

Being trait objects, this works for closures too.

```
fn invoke(f: FnOnce(A) -> B, a : A) -> B {
  f(a)
}
```

In practice, a decent optimizer should usually be able to monomorphize this for particular closure-implementing types where appropriate.

On the other hand, when returning a trait object from a function, something else interesting happens. Rather than execute the function to return the trait object, the whole
invovation expression is *lazily evaluated*. A conformant trait object is constructed that stores the necessary *environment* being passed in to the invocation, and the invocation
is only actually executed when a trait method gets called on the object. This is basically a closure in disguise. Though, keep in mind this is an implementation detail;
if an optimizer determines it can avoid the closure by reordering execution, it can do so.

```
fn invoke_a() {
  let m = new();
  // new does not actually get called until here, as part of the evaluation of a().
  m.a()
}
```

It is possible to *force* that a function immediately returns a trait object by using the *dyn* modifier in a return type. This modifier basically eliminates the laziness of trait objects.
Do note that this limits the possible ways to implement such a function. For now, the only way to return a dyn trait object from a function is if it came from one of the arguments (in which case,
we use the mentioned move implementation).

```
fn extract_trait(m: MyContainer) -> dyn MyTrait {
  m.m
}
```

In other contexts, *dyn* is unnecessary because it is actually the default for a trait object. Which implementation gets used by default depends on the context. In short,
function returns use lazy trait objects by default, and arguments and structs use dyn trait objects by default.

[Theory note: We are using lazy object to mean *a type with negative polarity*. For now we only have trait objects as examples of these. As long as we are limited
to trait objects, we are using *dyn* for the conversion to a type with positive polarity, i.e. the closure. We may add other means of creating these, but having this align with the trait object
system does give this concept some familiarity]

As in Rust, sometimes we do need to know that a trait object outlives some lifetime. Rust already allows us to specify this on traits, and as mentioned earlier, NewLang allows
this to be specified on any type. In some cases it can prove that a particular returned trait object, even if it does not explicitly specify it, must outlive a particular lifetime because of its inputs
and the lazy evaluation.
In such cases, it is legal to then add on the lifetime specifier to an object that did not previously have it. For dyn trait objects, this is more difficult to prove for reasons we will get to in the next section.

## Linear Types and Borrowed References

Before moving on to the meat of the next section, we need to get some prerequisites out of the way. Unlike Rust, NewLang is genuinely *linear-typed*. The `Drop` trait is used for all types that can be implicitly
dropped, not just those that explicitly need a custom implementation to drop resources. Most types are still expected to implement `Drop`, making them *affine*, but whether this is auto-derived like Rust's `Sync` or implemented by default like
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

Mathematics suggests that ideally, there are some laws we would want a `Susp`-like type to obey. `Susp` itself does not obey these, so I am tentatively calling this idealized type `Defer`. One law essentially amounts to lifting the above limitations on `.defer` - more explicitly it means that this type has a `map` operation
that can be used even when it is being borrowed. It is actually possible to implement this, but it does not come for free; one can use a linked list of callbacks whose nodes are on the stack. Writing and dropping a reference then requires traversing this list, which is a bit expensive for what should just be a "write" operation. Logically we would expect `Defer<T>` to behave like `FnOnce(FnOnce(T) -> ()) -> ()`. The other law is much more problematic - we can express it as the existence of a function `fn((Defer<()>, '0 % A)) -> A`. This says that once a value has been deferred to a `()`, we can actually hide it, even while it is still borrowed. This amounts to being able to ignore a lifetime under certain conditions. Implementing such a function under normal calling conventions is impossible. If we were to try to implement `Defer`, my current idea is to add yet another reference type, called `DMut<A, B>` - just as `Susp` is produced alongside `&tmut A/B`, `Defer` would be produced alongside `DMut<A, B>`. `Defer<B>` and `DMut<A, B>` would both be treated specially by NewLang (more precisely, via lack of some yet-to-be-named marker trait) - a function that returns `DMut<A, B>`, `Defer<B>`, or a type containing either one would actually be implemented using continuation-passing style rather than the normal stack - this should allow its order constraint to be forgotten if conditions are appropriate. Otherwise, `DMut<A, B>` is essentially the same as `&tmut A/B` and can itself be borrowed as the latter. (WIP: it may make sense to give `DMut<A, B>` more flexible unwinding options that `&tmut A/B`, in order to make it safe to catch.)

While mathematically we would want all order constraints to use `Defer`, in practice I believe the much cheaper `Susp` will cover most typical uses. `Defer` is probably necessary for some interesting control flow. `Defer` does establish what the most general semantics of borrowing are. `Susp` and `Pinned` can be seen as optimized special cases of `Defer`.

It may be possible to provide an extremely limited form of `.defer` to be used with `Pinned`. This seems to make sense when using a struct or enum constructor, and it may even be sound for non-side-effectful functions, like const functions. I am not sure if it is worthwhile to allow this, however.

An important note on how `defer` - or more precisely the `map` operation on `Susp`-like types - interacts with order constraints. Any resources that are used by a `defer` expression not only have their ownership taken over by the
resulting `Susp`-like instance, but also inherits any order constraints those resources had. Additionally, order constraints are considered transitive in NewLang, so even if we eliminate a `Defer<()>` in an order DAG, all constraints imposed transitively on the remaining values remain in place, re-associated with the remaining values. 

We now need to reconsider something fundamental. There is some expectation that a function `FnOnce(A) -> B` should be equivalent to at least one of the variations of `FnOnce(A, &out B) -> ()`. However, we just explained
how `&out B` has this deferred callback mechanism. To be most coherent, we want to say that `FnOnce(A) -> B` also has access to such a mechanism. Not for all `B` however! Lazily evaluated `B`, which for now means all trait objects, do not have the benefit of this mechanism, as the lazy evaluation mechanism works differently. For example, in `FnOnce(A) -> B -> C`, we cannot use a callback for `FnOnce(B) -> C`, but we *can* use one for `C`, provided it is not a lazy type. 

[Theory note: With this, we are basically saying that `FnOnce(&out T) -> ()` is the implicit conversion of a type with positive polarity into one with negative polarity.]

In fact, `&out B` where `B` is a trait object is actually implicitly a dyn trait object, as in function arguments.
Thus, we *do* get this mechanism for `FnOnce(A) -> dyn B`. This actually explains *how* we can return a dyn trait object from a function; we're not really returning it, but passing it to a provided callback. This
also gives us another means of creating a dyn trait object to return; we can borrow local variables inside the function. This is the reason why we cannot prove implicit *outlives* relationships on returned dyn trait objects.
Of course, we can always explicitly write them, and that puts restrictions on what we can borrow locally.

```
fn new2() -> dyn MyTrait {
  MyStruct { ... }
}
```

A minimal compiler implementation would be expected to optimize unused callbacks out, probably by monomorphizing on the callback.

This is all necessary in order to bypass a universal operational limitation of borrowing and in particular all `Susp`-like types (`Defer` included).
If the inner held type is not known to outlive any lifetime, then we have no way of statically tracking this type's lifetime across the `Susp` side boundary of a `Susp`/`&out` pair. This is an issue because we then cannot safely store the value until it is extracted; we have no choice but to completely use up this value in the callback at the time `&out` is consumed. We would be unable to implement a function `fn extract(d: Defer<T>) -> T` for all `T`.
This is where functions having an optional callback comes in; for all `T` that are not lazy types, then `extract` can be implemented by passing `T` to the callback. This happens implicitly; there is no need or ability to explicitly work with this callback. The language assembles the callback from the control flow of `T`, as it should be impossible for `T` to escape this callback
without outliving some lifetime.

```
fn extract_ref(d: Defer<&mut T>) -> &mut T {
  d
}
```

For lazy `T`, we cannot implement the type `fn extract(d: Defer<T>) -> T`, since such `T` do not use the callback mechanism. What we can do, however, is extract `dyn T`, which does use the mechanism. Again, there
is nothing we need to explicitly do for this to work, other than marking the types correctly.

```
fn extract_trait(d: Defer<MyTrait>) -> dyn MyTrait {
  d
}
```

To allow us to write a single generic `extract` function type, the return type of a function type can be marked with a `return` modifier. This forces the return type into a form that supports the callback mechanism;
for lazy `T`, this turns it into `dyn T`. Otherwise, it has no effect.

```
fn extract<T>(d: Defer<T>) -> return T {
  d
}
```

[Theory note: `return T` is actually explicitly forcing the conversion from a type of positive polarity to one of negative polarity. 
As part of that however, it also first implicitly converts `T` to a type of positive polarity if necessary.]

If using `Susp` instead of `Defer`, the usual caveats apply; the callback must be statically known. For `Pinned`, there is no callback so it is simply impossible to to store a trait object or reference without an explicit outlives specifier.

## Advanced .defer tricks

Whether `Susp` or `Defer`, we can use deferred callbacks to set up some tricks - more with the latter than the former. For example, we can borrow a `Box<A>` as an `&in A` and leave a `Susp<()>`. Internally, this works by setting up the deferred callback
to deallocate the box. Another example is, if we have some sort of channel, we could set up a reference whose callback automatically pushes to that channel. Many "Guard" types that combine a pointer with some sort of release
mechanism now become optional.

There is also another alternative to `Susp` or `Defer`. Any type on the "deferred" side of an order relationship can be placed into an `FnOnce` closure. Inside the closure the borrow is treated as ended, 
so provided that the inner value can be extracted when the borrow ends,
`FnOnce` can actually replace `Susp` or `Defer`.

```
let tmut x = 5;
let y: &tmut i32 = &tmut x;
let z = || x + 10;
*y *= 2;
z()
```

This works because, just as closures and trait objects have indefinite lifetimes due to being able to contain references, they also cannot escape being on the deferred side of an order constaint due to the possibility of containing a `Susp` or similar type. They then cannot be invoked until free of the order constraint - by which point the deferred value is guaranteed to be available.

Note that unlike with `Susp` or `Defer`, the closure and extensions will not be invoked as soon as the borrow ends. This is why it is necessary that the value inside the underlying `Susp` or `Defer` can be extracted
and stored.

[WIP: I am undecided what using defer syntax with an external variable inside a closure would mean]

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
  let g = || f(a);
  (g, outA)
}

fn from_parts<A, B>(f : Fn() -> B, outA : 'f % &out A) -> A -> B {
  |a| {
    *outA = a;
    f()
  }
}
```

We also should be able to implement this function:

```
fn extract_a(f: FnOnce(FnOnce(A) -> B) -> C) -> ('1 % A, FnOnce(B) -> C)
where C : 'static {
  let (result : Defer<(A, '0 % &out B)>, out_result) = DMut::borrow(());
  let c = f(|a| { let b; *out_result = (a, &out b); b });
  (result.0, |b| { *result.1 = b; c })
}
```

We need `C` to outlive some lifetime in order to ensure that it doesn't inherit the lifetime of `out_result`, which would prevent implementing the above code. As I don't currently have a syntax
for "outlives some lifetime", I am leaving it as "outlives `'static`" for now.

This function relies entirely on the deferred callback mechanism in order to be able to "return" a closure we constructed. The callback only ends once `FnOnce(B) -> C` is consumed.

This is the key that, slightly modified allows us to implement the following interface:

```
fn shift(f : FnOnce(FnOnce(A) -> B) -> C) -> ('1 % A, DLabel<B, C>) where C: 'static;
fn reset(d: ('1 % B, DLabel<B, C>>) -> C;
```

Some might recognize these names from literature on "Delimited continuations". That is the intention here - a linearly-typed variation of `shift/reset` APIs. Being linearly-typed does greatly restrict these compared
to the traditional versions, however, as well as being more awkward to use than in a dynamically-typed setting. In a way though, `Defer` and the implicit "return" callback are already a kind of delimited continuations; it can be conceptualized as `FnOnce(DMut<(), T>) -> ()`.

## Defer Streams

There is an interesting generalization of Defer to a *stream* of values. This looks something (but not exactly) like this:

```
type DStream<T> = Defer<DeferStreamInner<T>>;

enum DeferStreamInner<T> {
  Empty,
  Cons('1 % T, DStream<T>)
}
```

A DeferStream or `DStream<T>` lets you obtain one value of `T` at a time, but each value of a non-static `T` must be fully consumed before the next value can be obtained.
Furthermore, the entire stream is inside a `Defer` and subject to order constraints thereof. This is relevant because just as a `Defer` is usually created as a pair, so too is a `DeferStream`.

```
fn stream_example1() {
  let (stream, sink) = DStream::new();
}
```

A `DSink` can be conceived as a `DOut` generator, with constraints to ensure that only one `DOut` exists at a time.

For now, I will demonstrate how recursion can be used to consume a DStream.

```
fn consume(list: &mut Vec<T>, stream: DStreamInner<T>) {
  if let DStreamInner::Cons(t, ts) = stream {
    list.Add(t);
    consume(list, ts.defer);
  }
}

fn collect() -> Vec<i32> {
  let (stream, sink) = DStream::new::<int>();
  let v = Vec::new();
  consume(&mut v, stream.defer);
  sink.send(1);
  sink.send(2);
  drop(sink);
  v
}
```

Since DStream, containing `Defer`, cannot be a type known to outlive some lifetime, we have to use `defer` syntax explicitly to open up the stream despite having no order
constraint here.
We do not need to store the result of the `defer` because we are returning a `Defer<()>`, which can be eliminated. Note however that by transitivity,
the lifetime of `sink` then becomes constrained to the `v` borrowed in the `defer`, the latter then remaining borrowed until `sink` is consumed.

A DStream has some use in representing a list abstract while only using only a smaller buffer, but we can more or less achieve the same effect with a more traditional Stream/Iter interface by replacing `Defer` with `FnOnce`.
A caveat with how we presented this is that if `T` is static, then the `Defer` has no effect, and this is really no different from a list. Depending on what semantics one wants, this idea can be combined others to achieve more
useful semantics. A possible one is `Defer<FnOnce() -> T>`. The `FnOnce` gives an opportunity for last-minute computation that isn't associated with any borrowing, as in a traditional Stream.

The interesting part of DStream specifically is that it looks like an Event Stream, running a continuation as soon as a value is available. This is not enought to "implement" an Event Stream; the "events" here are just function calls with a statically-ordered control flow, not a true dynamic event system. However, an Event Stream could be presented as a `DStream` interface by telling the underlying event handler to "feed" a `DSink`. Still, DStream
is a blocking interface; for real-world applications, one may want to combine these ideas with some sort of non-blocking mechanism. That is outside the scope of the current section, however.

There is another interesting type associated with DStream; it can be defined as roughly `FnOnce(DStream<&out T>)`. We call this type `DSignal<T>`, in correspondance with terminology from some versions of Functional Reactive Programming (classic FRP calls this a "Behavior"). A
`DSignal<T>` must be able to provide a value of `T` on demand as a continuation of some event, as many times as demanded. Because these events occur at monotonically increasing points in time, a `DSignal<T>` is a potential model for a time-varying function returning `T`; this is the interpretation of a "Signal" in Functional-Reactive Programmming.

Note that *function* here really means an `FnMut`; a function that could only be called once would not be a useful Signal. Accepting a `DStream` of requests means that this time-varying function can be called
multiple times, as long as the time in question is monotonically increasing.

Signals are interesting, in part because they present an immutable interface over something that is very much mutable. My hypothesis, which I have not verified, is that through duplexing,
it is possible to safely implement `Clone` for Signals. 

My hypothesis is that linearity and order constraints would allow for presenting an interface close to classic/continuous FRP but that disallows space leaks.

`DSink<T>` may be logically equivalent to `DSignal<&out T>`.

DStream can be combined with complex types to achieve protocols similar to session types. For example, a server taking requests from a `DStream<&tmut Request/Response>` is given both a Request and a place to send a response, which is is obligated to send before taking the next request. The corresponding client sees this as a type `DSignal<FnOnce(Request) -> Response>`.

## [Skipping a bit]

## Side Effects

From here on out, things will diverge rapidly from Rust. While I have been using Rust syntax here and will continue to do so for now, NewLang was originally conceived as a functional programming language inspired by Haskell.
One element of that is control over the use of side effects. For our purposes, we only care about *observable* side effects. Modern Haskell uses something called monads to control side effects, but there is a related language called Clean that instead uses something called uniqueness types, which are similar to linear types. Our approach is based on the latter. In NewLang, the consumption and return of linear/affine resources, or the mutation of a mutably borrowed resource, are *not* considered side effects. Our goal is that all observable side effects occur through such linear/affine resources. A function that can perform such a side effect will have to reflect in its type that it borrows the corresponding resources - in this way, these resources are essentially capabilities.

The primary such resource type `IO`. This can only safely be obtained when writing the `main` method for an application.

```
fn main(mut io: IO) {
  ...
}
```

All IO actions ultimately begin as methods on this io resource.

```
fn main(mut io: IO) {
   // Exact API is just for demonstration; no decisions have been made on specific APIs
   io.println("Hello, World!");
}
```

`println` here is a method that mutably borrows a resource of type `IO`.

Some methods may return another resource independent of the `IO` value. This is especially appropriate for anything representing an OS handle of some kind.

```
fn main(mut io: IO) {
  let file = io.open("/myfile.txt").unwrap();
  let text = file.read();
  io.println("read file");
  text = text + "and the end";
  file.write(text);
}
```

This `IO` value or a derived resources need to be passed to any function that wants to do IO. This may be inconvenient, but as long as a single `IO` value is passed around, a program should be deterministic in a sense; while we cannot control the state of the system that comes into the program, the program should execute on a given state in a predictable way.

Of course in the real world, not every program can be written this way. If you have a multiprocess or distributed application, each process or node will need access to IO just to communicate any coordination. To allow for such scenarios, `IO` implements a trait called `Duplicate`.

```
trait Duplicate {
  fn duplicate(self: Self) -> (Self, Self);
}
```

`Duplicate` at first seems conceptually the same as Rust's `Clone`. However, whereas `Clone` borrows a value immutably and produces copy equal to the borrowed value, `Duplicate` consumes a resource and produces two new resources
that are supposed to be equal in some sense. The consumption of a resource here is important! We don't make any guarantees that the resulting resources are equal to the one that was consumed. That allows us to say that the very
act of duplication has observable side effects, which `Clone` does not allow.

This is important for `IO`, because it lets us play an interesting trick. Through the act of duplication, it is now conceptually part of the `IO` resource's "state" that there is another `IO` resource floating around, potentially
changing the effect of all future operations on our `IO` resource. It just happens that the specific "state" returned by `duplicate`, and thus the specific future effects, are non-deterministic. This is ok though; we can see this non-determinism itself as being part of the state of `IO`. As long as the language perceives that the resource is being mutated, effects that can be abstracted as the result of this mutation are acceptable.

While there is nothing wrong with it from a language perspective, duplicating an `IO` resource does mean that the user must accept non-determinism from their program, or at least take steps to control it. The *only* guarantee that NewLang
makes about order of effects is that calls on an individual resource instance are guaranteed to occur sequentially. Beyond that,
there is *no* guarantee that calls on different `IO` resource instances will occur in a given order or not at the same time. In fact, a valid
NewLang compiler would be free to silently re-order such calls. Atomicity is up to individual APIs.

```
fn main(io: IO) {
  let (mut io1, mut io2) = io.duplicate();
  io1.println("Foo");
  io2.println("Bar");
  // This may print "FoBoar" and still be considered valid behavior.
}
```

For scenarios where we cannot allow the compiler to reorder calls, the most straightforward means of forcing an order is to collect all the resources into a tuple a borrow from that.

```
fn main(io: IO) {
  let mut tpl = io.duplicate();
  tpl.0.println("Foo");
  tpl.1.println("Bar");
  // This can only print "FooBar".
}
```

Beyond simple static scenarios, there are a variety of techniques that people use to coordinate dynamically across larger scenarios. The most common one is the use of "locking" - code that wishes
to borrow a shared resource must wait until no one else is borrowing it. Another popular pattern is to grant exclusive ownership of a resource to some kind of service, and all interaction with that resource
must occur through messaging that service. A less well known family of techniques are "optimistic" - code assumes it has access, and if that assumption turns out to be wrong it must rollback any effects before committing them.

All of these dynamic techniques still allow for some non-determinism. Somewhere in their implementation is a place with multiple concurrent writers, often adding onto a queue of some kind. The need for shared mutable resources
cannot be fully avoided. `Duplicate` is our tool for working with this.

Let us look more closely at locking. In Rust, one shares a reference to a mutex, `&Mutex<T>`, and after waiting can obtain access to `&mut T`. In NewLang this is problematic; an observable effect can occur even though we started with an immutably borrowed reference. Rust allows this with interior mutability, but NewLang promises that effects cannot occur through nonlinear values.

We instead introduce a new non-copy but duplicable borrowed reference type, `&dup`. Effectful APIs can be built on `&dup T` in much the same way as for `IO`. So for example, `&dup Mutex<T>` in NewLang takes the place of `&Mutex<T>` in Rust mutex APIs. `&Mutex<T>` is of little use in NewLang since it cannot be used with effectful APIs.

In Rust it is common to put a `Mutex` inside of an `Arc` pointer. In NewLang we need to be careful of this, however; an `Arc` is supposed to have cloneable semantics, but an `Arc<Mutex<T>>` is also effectful, which is problematic.

[WIP]
We have gone through several iterations for how to handle this, but the current solution is to have a `Pure` marker trait, which essentially says that `&` and `&dup` are the same for a type. Any type without interior mutability
automically implements this, much like the recently introduced `Freeze` trait in Rust. Additionally, a user can unsafely mark a type as being `Pure`, promising that any interior mutability is not externally visible. If `T` is
`Pure`, the `Arc<T>` can safely impement `Clone`. Otherwise, `Arc<T>` is merely `Duplicate`, meaning that `&Arc<T>` cannot be used to produce additional references. In either case, `&Arc<T>` provides access to `&T`, and `&dup Arc<T>` provides access to both `&T` and `&dup T`. This way, `Arc<Mutex<T>>` is treated as a resource.

Incidentally, some Rust APIs on `Arc` directly, such as those exposing the ref count or `Weak` pointers, are considered effectful as they change according to what independent references do.
For that reason, NewLang cannot expose the APIs directly on `Arc`. As long as this problem remains limited - it should be only shared smart pointers presenting a pure abstraction but exposing internal state - our best plan is to also provide an `ImpureArc` type, basically a wrapper with an unsafe constructure
that always disables `Clone`, allowing for these effectful APIs to be exposed.

A mutex could also be placed inside of a static variable. In Rust, that can be used via `&`, but in NewLang we need to use it through `&dup`. A function can use a static variable through `&dup`, but that has implications;
the function's environment, even if it is a top-level function, is now considered to be merely duplicable instead of copyable. We need a new trait, `FnDup`, to handle closures of this kind.

In order to allow interior mutation to be used as an implementation detail, it is possible to *unsafely* go between `&` and `&dup`. 
[WIP:
We may want to create a new *impure* modifier just for this purpose without other unsafe semantics.
]

Moving down to single-threaded, we can do the same for `Rc`, `RefCell`, and `Cell`.

In practice, it is more convenient if `Duplicate` is defined instead as cloning through a `&dup`, with `duplicate` being defined in terms of that.
