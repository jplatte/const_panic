[![Rust](https://github.com/rodrimati1992/const_panic/workflows/Rust/badge.svg)](https://github.com/rodrimati1992/const_panic/actions)
[![crates-io](https://img.shields.io/crates/v/const_panic.svg)](https://crates.io/crates/const_panic)
[![api-docs](https://docs.rs/const_panic/badge.svg)](https://docs.rs/const_panic/*)


For panicking with formatting in const contexts.

This library exists because the panic macro was stabilized for use in const contexts
in Rust 1.57.0, without formatting support.

All of the types that implement the [`PanicFmt`] trait can be formatted in panics.

# Examples

- [Basic](#basic)
- [Custom Types](#custom-types)

### Basic

```compile_fail
use const_panic::concat_panic;

const FOO: u32 = 10;
const BAR: u32 = 0;
const _: () = assert_non_zero(FOO, BAR);

#[track_caller]
const fn assert_non_zero(foo: u32, bar: u32) {
    if foo == 0 || bar == 0 {
        concat_panic!("\nneither foo nor bar can be zero!\nfoo: ", foo, "\nbar: ", bar)
    }
}
```
The above code fails to compile with this error:
```text
error[E0080]: evaluation of constant value failed
 --> src/lib.rs:17:15
  |
8 | const _: () = assert_non_zero(FOO, BAR);
  |               ^^^^^^^^^^^^^^^^^^^^^^^^^ the evaluated program panicked at '
neither foo nor bar can be zero!
foo: 10
bar: 0', src/lib.rs:8:15
```

### Custom types

Panic formatting for custom types can be done in these ways
(in increasing order of verbosity):
- Using the [`impl_panicfmt`] macro (requires the default-enabled `"non_basic"` feature)
- Using the [`flatten_panicvals`] macro (requires the default-enabled `"non_basic"` feature)
- Manually implementing the [`PanicFmt`] trait as described in its docs.

This example uses the [`impl_panicfmt`] approach.

```compile_fail
use const_panic::concat_panic;

const LAST: u8 = {
    foo_pop(Foo{
        x: &[],
        y: Bar(false, true),
        z: Qux::Left(23),
    }).1
};


/// Pops the last element in `foo.x`
///
/// # Panics
///
/// Panics if `foo.x` is empty
#[track_caller]
const fn foo_pop(mut foo: Foo<'_>) -> (Foo<'_>, u8) {
    if let [rem @ .., last] = foo.x {
        foo.x = rem;
        (foo, *last)
    } else {
        concat_panic!(
            "\nexpected a non-empty Foo, found: \n",
            // uses alternative Debug formatting for `foo`,
            // otherwise this would use regular Debug formatting.
            alt_debug: foo
        )
    }
}

struct Foo<'a> {
    x: &'a [u8],
    y: Bar,
    z: Qux,
}

// You need to replace non-static lifetimes with `'_` here.
const_panic::impl_panicfmt! {
    impl Foo<'_>;

    struct Foo {
        x: &[u8],
        y: Bar,
        z: Qux,
    }
}

struct Bar(bool, bool);

const_panic::impl_panicfmt! {
    impl Bar;

    struct Bar(bool, bool);
}

enum Qux {
    Up,
    Down { x: u32, y: u32 },
    Left(u64),
}

const_panic::impl_panicfmt!{
    impl Qux;

    enum Qux {
        Up,
        Down { x: u32, y: u32 },
        Left(u64),
    }
}
```
The above code fails to compile with this error:
```text
error[E0080]: evaluation of constant value failed
  --> src/lib.rs:112:5
   |
7  | /     foo_pop(Foo{
8  | |         x: &[],
9  | |         y: Bar(false, true),
10 | |         z: Qux::Left(23),
11 | |     }).1
   | |______^ the evaluated program panicked at '
expected a non-empty Foo, found:
Foo {
    x: [],
    y: Bar(
        false,
        true,
    ),
    z: Left(
        23,
    ),
}', src/lib.rs:7:5

```

# Limitations
#![doc = crate::doc_macros::limitation_docs!()]

# Cargo features

- `"non_basic"`(enabled by default):
Enables support for formatting structs, enums, and arrays.
<br>
Without this feature, you can effectively only format primitive types
(custom types can manually implement formatting with more difficulty).

# Plans

Adding a derive macro, under an opt-in "derive" feature.

# No-std support

`const_panic` is `#![no_std]`, it can be used anywhere Rust can be used.

# Minimum Supported Rust Version

This requires Rust 1.57.0, because it uses the `panic` macro in a const context.


[`PanicFmt`]: https://docs.rs/const_panic/*/const_panic/fmt/trait.PanicFmt.html
[`impl_panicfmt`]: https://docs.rs/const_panic/*/const_panic/macro.impl_panicfmt.html
[`flatten_panicvals`]: https://docs.rs/const_panic/*/const_panic/macro.flatten_panicvals.html