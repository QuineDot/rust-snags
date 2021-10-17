# Rust Snags

Gotchas and warts in Rust Lang (IMHO).

## Lifetime elision
* [Closures use different rules than functions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#10-closures-follow-the-same-lifetime-elision-rules-as-functions)
  * This makes creating higher-ranked closures arduous
  * So much so that many think it's impossible to make closures that are generic over liftimes
  * Outlook: Aparently backwards incompatible to change?  Not sure why though. Could be largely fixed with some new syntax, e.g. the ability to ascribe an `impl Fn(&Foo)` type.
* Struct lifetime parameter elision
  * By which I mean eliding the parameter altogether.  [Most seem to agree this is non-obvious and bad style.](https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html#example-type-annotations)
  * Part of [RFC 0141](https://rust-lang.github.io/rfcs/0141-lifetime-elision.html)
  * Can also be a gotcha as an associated type (`'static)` vs return position (can default to input lifetime)
  * Will likely be a gotcha when defining a GAT
  * Alternative: Use the anonymous lifetime (`<'_>`, [RFC 2115](https://rust-lang.github.io/rfcs/2115-argument-lifetimes.html))
  * Outlook: Stable, but could be deprecated over time
* `dyn Trait` lifetime elision
  * [RFC 0599](https://rust-lang.github.io/rfcs/0599-default-object-bound.html), [RFC 1156](https://rust-lang.github.io/rfcs/1156-adjust-default-object-bounds.html), [RFC 1214](https://rust-lang.github.io/rfcs/1214-projections-lifetimes-and-wf.html)
  * More of a gotcha than a wart; not a problem if your lifetime of applicability is `'static`
  * The elision rules are rather special-cased
  * The elision rules sometimes choose the wrong lifetime
  * The elision rules naturally do not penetrate aliases... including `Self` or associated types
  * [In generic struct declarations](https://rust-lang.github.io/rfcs/2093-infer-outlives.html#trait-object-lifetime-defaults) it's based on the underlying type constraints
    * Which enforces incorrect impression that the lifetime is that of the underlying type
    * Probably too obscure to really be a snag people actually hit
  * Outlook: Stable and probably not changeable
* "In-band" lifetimes
  * I.e. just using a named lifetime declares it (if not already declared)
  * Part of [RFC 2115](https://rust-lang.github.io/rfcs/2115-argument-lifetimes.html)
  * Optimizes for writing, pessimizes reading, foot-gun
  * Outlook: Approved but not yet stable.  Divisive?

## `impl Trait`
* Argument Position `impl Trait` (APIT)
  * Some love it, some hate it.  Distinguishing from RPIT is definitely a stumbling block to beginners
  * Part of [RFC 1951](https://rust-lang.github.io/rfcs/1951-expand-impl-trait.html)
  * Outlook: Divisive but stable and unlikely to change
* [`impl Trait` "leaks" auto-traits](https://rust-lang.github.io/rfcs/1522-conservative-impl-trait.html#semantics)
  * Probably only a gotcha in that it's easy to break backwards compatibility
  * Outlook: Stable, by design, and probably can't change
* The "captures" gotcha
  * [Described here](https://github.com/rust-lang/rust/issues/34511#issuecomment-373423999) and [explored here](https://users.rust-lang.org/t/lifetimes-in-smol-executor/59157/8?u=yandros)
  * See also: [Crate work-around](https://docs.rs/fix-hidden-lifetime-bug/0.2.4/fix_hidden_lifetime_bug/), [`async` issue](https://github.com/rust-lang/rust/issues/63033)
  * Outlook: Something like `dyn Trait`'s lifetime intersection [is desired]( https://github.com/rust-lang/rust/pull/57870#issuecomment-459116559) but has not yet materialized.  Rust teams unwilling to stabilize a work-around in the meanwhile.

## `dyn Trait`
* [Lifetime variance and unsized coercion interaction](https://users.rust-lang.org/t/solved-variance-of-dyn-trait-a/39733)
  * The lifetime is covariant and that covariance applies _during unsized coercion,_ which can bypass otherwise invariant contexts such as inside an `UnsafeCell`
  * Surprising but I'm not sure if it's even a gotcha to be honest.  Obscure.  
  * Outlook: [stable](https://rust-lang.github.io/rfcs/0599-default-object-bound.html#detailed-design) and unlikely to change
* `where Self: Sized` as a coarse approximation for `where Self: !Dyn`
  * Precludes implementing a function not only for `dyn ThisTrait`, but `dyn AnyTrait`, `str`, `[T]`, ...
  * Leads to situation where the only implementation for unsized types are default function bodies (need to re-find the citation for this situation)
  * Seems to be special-cased in the compiler (citation needed)
  * Outlook: Used endemically but a replacement should be possible
* [Inherent methods "are" the trait implementation](https://github.com/rust-lang/rust/issues/51402)
  * Striving too hard for `dyn Trait` "is" the trait vs. the reality: `dyn Trait` is a distinct, concrete type
  * Related: The [belief/desire](https://github.com/rust-lang/rust/issues/88904) for it to be impossible to make a `dyn Trait` that doesn't implement `Trait`
    * See also [RFC 2027](https://rust-lang.github.io/rfcs/2027-object_safe_for_dispatch.html)
  * Outlook: Stable, but could probably just be allowed?

## Variance
* Traits are invariant
  * This is [contrary to the RFC](https://rust-lang.github.io/rfcs/0738-variance.html#phantom-functions) and was [yanked with no in-band discussion](https://github.com/rust-lang/rust/pull/23938) really
  * Leads to somewhat painful variance-like implementations or helper methods
  * But arguably mostly a documentation / RFC follow-up issue
  * Outlook: Could be brought back but not on the horizon
* GATs are invariant
  * Leads to somewhat painful variance-like implementations or helper methods
  * Outlook: GATs are stable yet.  Otherwise, same as for traits?

## Default type paramaters
* Default type parameters are just generally a half baked feature
  * [Doesn't interact with inference](https://github.com/rust-lang/rust/issues/36980#issuecomment-251726254)
  * [RFC 213](https://github.com/rust-lang/rust/issues/27336) basically cancelled
  * [Underdocumented](https://github.com/rust-lang/reference/issues/24)
  * Outlook: Could improve but no momentum to do so
* `ControlFlow` parameter order
  * [The order was changed and a default added](https://github.com/rust-lang/rust/pull/76614) back when the primary use was iterator combinators
    * Optimizing for writing via the default parameter
  * [The MCP to expand its usage](https://github.com/rust-lang/compiler-team/issues/374) happened afterwards
    * That includes the `Try`/`?` operator
  * [Now it's confusing that the order is the opposite of `Result`](https://github.com/rust-lang/rust/issues/84277#issuecomment-907237889)
  * Outlook: Stabilized and we're stuck with it forever

## Ownership
* `_` does not bind
  * So `let _ = ...` is different from `let _name = ...`
  * [Which often makes whatever it matches into a temporary](https://github.com/rust-lang/rust/issues/10488#issuecomment-30879810)
  * Outlook: Permanent.  It supplies unique abilities in some cases.
* Drop glue determines owned lifetime (drop at end of scope)
  * [Despite NLL](https://github.com/rust-lang/reference/issues/873#issuecomment-768951633), which can be surprising
  * Especially in the face of `stdlib` types utilizing [RFC 1327](https://rust-lang.github.io/rfcs/1327-dropck-param-eyepatch.html) (`#[may_dangle]`)
  * Outlook: Probably permenant?  [`#[may_dangle]` stabilization](https://doc.rust-lang.org/nomicon/dropck.html#an-escape-hatch) would help some
* [NLL problem case #3](https://github.com/rust-lang/rust/issues/51545)
  * Outlook: Waiting on Polonius

## Misc
* Fallback integer is `i32` but fallback float is `f64`
  * Outlook: Permenant
* [Partial implementations](https://github.com/rust-lang/rust/issues/31844) as implemented on nightly (part of Specialization)
  * Recycles the name `default` for `default impl` because this is "less confusing" than calling them `partial impl`
  * Makes `default` implicit on the implementations within
  * Thus they are not finalizable
  * Outlook: Not stable and considered an unresolved question
* Lack of Needle API ([RFC 2500](https://rust-lang.github.io/rfcs/2500-needle.html))
  * There is no replacement other than the `bstr` crate for `[u8]`
  * [RFC 2295](https://github.com/rust-lang/rust/issues/49802) may help with `OsString`
  * Outlook: could be revamped but momentum is lacking
* Closures capture entire variables, not fields
  * Outlook: [RFC 2229](https://rust-lang.github.io/rfcs/2229-capture-disjoint-fields.html) will stabilize in edition 2021
* :sparkles: Self referential structs :sparkles:

## Mostly just lacking in documentation
* Inference and coercion order
  * [Here's an example](https://github.com/rust-lang/rust/issues/89299) of inference and coercion interacting for a surprising result.
* `dyn Trait` generally
* `&mut -> &` function semantics
  * I.e. [the returned borrow is still exclusive](http://smallcultfollowing.com/babysteps/blog/2018/11/10/after-nll-moving-from-borrowed-data-and-the-sentinel-pattern/#permissions-in-permissions-out)
* How functions capture more generally (e.g. complicated borrowing parameters)
  * By which I mean, returning something borrowed extends all input borrows
* How the `.` operator works
  * Including field accesses, not just method resolution
