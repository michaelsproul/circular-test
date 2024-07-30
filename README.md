# circular dep with optional features

This is a minimal reproduction of some unfortunate cargo behaviour involving optional dependency
features.

Our set up involves two crates `lib1` and `lib2`.

- `lib1` depends on `lib2`
- `lib2` _optionally_ depends on `lib1`
- `lib2` has a feature called `not-circular` which enables the `not-circular` feature on `lib1` using
  `lib1?/not-circular` syntax.
- `lib2` has a feature called `circular` which turns on its optional `lib1` dependency. Turning this
   feature on is expected to break.

The docs have this to say about the `?` syntax:

> The "package-name/feature-name" syntax will also enable package-name if it is an optional dependency. Often this is not what you want. You can add a ? as in "package-name?/feature-name" which will **only enable the given feature if something else enables the optional dependency**.

One might interpret this to mean that as long as nothing enables the `lib1` dependency of `lib2`, that
we should be able to compile with the `not-circular` feature of `lib2` and have it _not enable_ the
`not-circular` feature of `lib1`.

However, what currently happens (Rust 1.80.0) is that Cargo seems to detect the existence of `lib1`
in the dependency tree, **takes this to mean that `lib1` is enabled**, and then detects a cycle.

This is reproduced by the following from within `./lib1`:

```
cargo check --features lib2/not-circular
```

```
error: cyclic package dependency: package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)` depends on itself. Cycle:
package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
    ... which satisfies path dependency `lib1` (locked to 0.1.0) of package `lib2 v0.1.0 (/home/michael/Programming/circular-test/lib2)`
    ... which satisfies path dependency `lib2` (locked to 0.1.0) of package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
```

Compiling the `lib2` crate has a similar effect:

```
cargo check --features not-circular
```

```
error: cyclic package dependency: package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)` depends on itself. Cycle:
package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
    ... which satisfies path dependency `lib1` of package `lib2 v0.1.0 (/home/michael/Programming/circular-test/lib2)`
    ... which satisfies path dependency `lib2` of package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
```

However, more oddly, `lib2` doesn't compile even when the `not-circular` feature is turned off:

```
cargo check
```

```
error: cyclic package dependency: package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)` depends on itself. Cycle:
package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
    ... which satisfies path dependency `lib1` of package `lib2 v0.1.0 (/home/michael/Programming/circular-test/lib2)`
    ... which satisfies path dependency `lib2` of package `lib1 v0.1.0 (/home/michael/Programming/circular-test/lib1)`
```

I don't understand why this is, because `lib2` compiles just fine when it is compiled as dependency of `lib1`. Does cargo treat the top-level crate being compiled differently wrt dependencies? Is it implicitly checking that all features are valid?
