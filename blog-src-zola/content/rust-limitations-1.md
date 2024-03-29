+++
title="Rust - Limitations - 1"
description="Of course, it's not yet done, and its's hard to design perfect things."
date=2023-12-29
updated=2024-01-29

[taxonomies]
tags = ["rust-language", "rust-limitations"]
categories = ["rust"]

[extra]
ToC=true
+++

# Reborrowings Are NOT After All Possible Type Inferences

'Reborrowing' of a reference, as a type of coercion, needs the information of
the 'source' and 'destination' type. But it's not the case that a reborrowing
only happens after all type inferences are done, so in some cases, when the type
information isn't complete, reborrowing does not happen as one would expect:

```rust
let mut n: i32 = 42;
let r1 = &mut n;

// Works, it's known that `r2` is a mutable reference, a reborrow take
// places here:
let r2: &mut _ = r1;
// Error, `r1` is moved here because reborrow isn't triggered when the
// type of `r2` is not known:
let r2 = r1;
// Also works, here it is also known that `r2` is a mutable reference:
let r2 = &mut *r1;

*r1 = 43;
```

```rust
fn func<T>(a: T, b: T) {}
fn test<T>(a: &mut T, b: &mut T) {
    func(a, b);
    // Error: `a` is moved. But we are free to use `b`:
    func(a, b);
}
```


# Lifetimes of Closures

Closures are nothing but a auto-generated `struct` that auto-implements special traits
(`FnOnce`, `FnMut`, `Fn`) that have a special `call` method, you can think of these
traits defined as:

```rust
trait FnOnce<Arg> {
    type Output;
    fn call(self, arg: Arg) -> Self::Output;
}

trait FnMut<Arg>: FnOnce<Arg> {
    fn call(&mut self, arg: Arg) -> <Self as FnOnce>::Output;
}

trait Fn<Arg>: FnMut<Arg> {
    fn call(&self, arg: Arg) -> <Self as FnOnce>::Output;
}
```

These definations are accurate enough for our discussion into the limitations
of Rust's closures.

## Argument Lifetime Problems

### For All vs For Specific

As the following example shows, a closure can work for a wide range of
arguments or a specific type of arguments:

```rust
struct Captured;

// `Captured` implements `FnMut<&'a str>` for all `'a`, i.e.
// the `call` signature is like `Captured.call<'a>(&'a mut str)`, it
// can be used for any `&mut str`
impl<'arg> FnMut<&'arg mut str> for Captured {
    type Output = ();
    fn call(&mut self, arg: &'arg mut str) -> Self::Output { }
}

// Let's call the lifetime of a `Captured` `'captured`, then
// here `Captured` only implements `FnMut<&'captured mut str>` for
// a specific `'captured`:
impl FnMut<&'captured mut str> for Captured {
    type Output = ();
    fn call(&mut self, arg: &'captured mut str) -> Self::Output { }
}
```

Unfortunately, the compiler can not determine which one is needed for each
and every case, and use a rather basic rule to decide:
If the argument is type annotated, it is the first case, otherwise, it is the
second case:

```rust
// `r` infered to have a specific lifetime:
let f1 = |r| {};
// `r` infered to have a higher kinded (any) lifetime:
let f2 = |_r: &mut i32| {};
// What if you just want the type annotatation, but not the
// higher kinded lifetime?
// One workaround is like this:
let f3 = |r| { let r: &mut i32 = r; }

let mut n = 42;

// Doesn't work, the first borrow lives as long as the `f1` lives:
f1(&mut n);
f1(&mut n);

// Works:
f2(&mut n);
f2(&mut n);
```

## Return Type Limitation

Without special handling, the return type of a closure might only use lifetimes concerning
the closure type itself:

```rust
// One might think this is equivalent to `fn f1<'a>(&'a mut i32) -> &'a mut i32`,
// but it's actually `fn f1<'a>(&'a mut i32) -> &'captured i32`, so the compiler
// complains that `'a` should outlive `'captured`:
let f1 = |x: &mut i32| x;

struct Captured;
impl<'arg> FnMut<&'arg str> for Captured {
    // The compiler uses `'captured` even though `'arg` is usable.
    type Output = &'captured str;

    // And, notably, it's also impossible to use `'lself` defined here.
    fn call<'lself>(&'lself mut self, arg: &'arg str) -> Self::Output {
        arg
    }
}
```

## Workarounds

- Use the type system to give hints to the compiler:
  ```rust
  fn enforce<F: for<'a> Fn(&'a mut i32) -> &'a mut i32>(f: F) -> F { f }
  let mut n = 42;
  let f = enforce(|x| { x });
  f(&mut n);
  f(&mut n);
  ```
- Use the [closure lifetime binder (not stable yet)](https://rust-lang.github.io/rfcs/3216-closure-lifetime-binder.html):
  ```rust
  let mut n = 42;
  let f = for<'a> |x: &'a mut i32| -> &'a mut i32 { x };
  f(&mut n);
  f(&mut n);
  ```
- Use functions, if the closure captures things, pass them as parameters or
  collect them into your own type.

# References

- [Reference - Type Coercions](https://doc.rust-lang.org/reference/type-coercions.html#coercion-types)
- [Issue 100002](https://github.com/rust-lang/rust/issues/100002)
- [Issue 41078](https://github.com/rust-lang/rust/issues/41078#issuecomment-293646723)
- [Error E0521](https://doc.rust-lang.org/stable/error_codes/E0521.html#error-code-e0521)
- [Why does the type annotation incur the reborrow?](https://users.rust-lang.org/t/why-does-the-type-annotation-incur-the-reborrow/88479/10)
- [Why closure cannot return a reference to data moved into closure](https://users.rust-lang.org/t/why-closure-cannot-return-a-reference-to-data-moved-into-closure/72655)
