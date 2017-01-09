- Feature Name: hygiene
- Start Date:
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Hygiene for macros by example 2.0.

# Motivation
[motivation]: #motivation

## Names in macro definitions behave reliably

More specifically, names in macro definitions behave independently of
 - where the macro is invoked (i.e. in which crate/module/scope)
 - what names are used in the macro invocation arguments.

### Reliably use items accessible from the macro definition

Today, the only way a macro author can reliably use an item from outside the macro definition
is to employ a `$crate::`-prefixed path.

This is cumbersome and doesn't allow macro authors reliably use
 - trait items (since the trait methods in scope are not reliable), or
 - anyting not accessible from outside the crate.

For example, assuming the following hygienic macro `foo::m` is invoked in a statement context,
```rust
fn f() {} // (i)
pub mod foo {
    fn g() {} // (ii)
    mod bar {
        pub fn h() {} // (iii)
    }
    trait T {
        fn f(&self) {} // (iv)
    }
    impl T for () {}

    pub macro m() {
        use super::f; // This always imports (i)
        super::f(); // This always resolves to (i)
        g(); // This always resolves to (ii)

        use self::bar::*; // This always imports (iii)
        h(); // This always resolves to (iii)

        ().f(); // This method call always resolves to (iv)
    }
}
```

### Reliably define and use items/fields/lifetimes/etc. in a macro definition

Today, a macro author cannot even reliably define and use an item;
it is always possible for a macro user to make the item to trigger a conflict error.

For example, assuming the following hygienic macro `m` is invoked in a statement context,
```rust
macro m($i:ident) {
    fn f() {} // (i) This is never a conflict error
    f(); // This always resolves to (i)

    mod foo {
        pub fn $i() {} // (i)
        pub fn f() {} // (ii) This is never a conflict error
    }
    foo::$i(); // This always resolves to (i)
    foo::f(); // This always resolves to (ii)

    fn g<$x, T /* this is never a conflict error */>(x: ($x, T)) {}

    struct S {
        $i: u32,
        x: i32, // This is never a conflict error
    }

    impl S {
        fn $i(&self) -> u32 { 0 }
        fn f(&self) -> i32 { 0 } // This is never a conflict error
    }

    let s = S { x: 0, $i: 0 };
    (s.$i, s.x); // This always has type (u32, i32)
    (s.$i(), s.f()); // This always has type (u32, i32)
}
```

## Allow macro authors to use macro arguments in more places

For example, consider
```rust
macro m($it:item, $t:ty) {
    mod foo {
        $it
        fn f(_: $ty) {}
    }
}

fn f() { ... } // If this is a valid item
fn g(_: Vec<...>) {} // and `Vec<...>` is a valid type,
m!(fn f() { ... }, Vec<...>); // then this invocation should compile.
```

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

- How is this idea best presented—as a continuation of existing Rust patterns, or as a wholly new one?

  As macros 2.0 is supposed to replace today's macros, its best if we teach them from scratch without reference to the old system.

- What names and terminology work best for these concepts and why?

  Little new terminology over macros 1 is needed.
  Hygiene is the principle that renaming things doesn't change their meaning---previous valid programs might become invalid if too many identifiers are renamed the same,

  The standard programming language terminology is thankfully not too exotic sounding in this case, but does make some distinctions not popularly observed.
  A classic example is *identifier* vs *variable*: an "identifier" is a very crude concept, and is well defined even for simple "token trees" where there is no notion of binding even, while a variable has more semantic baggage and

- Would the acceptance of this proposal change how Rust is taught to new users at any level?

  *Defining* macros would remain a skill unneeded by beginners just as it is today.
  There are some macros that even beginers *use* (panicking, printing, etc), but as they return an expression they are easy to use without understand the macro system.
  This would also not change.

- What additions or changes to the Rust Reference, _The Rust Programming Language_, and/or _Rust by Example_ does it entail?

  Any existing documentation of the old macro system would be adapted for this.

- How should this feature be introduced and taught to existing Rust users?

  The key virtue of hygiene is one can completely resolve names (decide whether an identifier is a fresh binding or resolves to another binding sites looking at the macro body alone.
  This sounds fancy, but is no different on how normal rust is lexically scoped---identifiers in the function are resolved regardless of how it is called.

  Doing this is actually fairly simple.
  If the identifier looks different—e.g. rust variable `foo` vs "$-prefixed macro variable `$foo`—then it must resolve separately.
  In fact, one can pretend that "$" were allowed in normal rust identifiers, and read the macro body as normal Rust code—the resulting name resolution is entirely that!

  Only when reasoning about macro *invocations* (as opposed to definitions) is extra knowledge unavoidable.
  The overriding principle is one can up-front reason about any identifiers usable across the expansion site without knowing the details of the macro's definition.
  Only identifiers passed in or bound in both over *both* expansion site and definition site can be used to "communicate" between the macro expansion and the code surrounds it.
  We can break this "communication" down and treat each direction separately.
  In the "out" direction, identifiers only bound over or inside the macro definition will never "leak" out of the expanded syntax and become in scope around the expansion site
  In the "in" direction, identifiers only bound over the macro invocation site and not explicitly passed in will never leak into the expanded syntax and become in scope inside it.

  An interesting aspect of the above however is its possible to "smuggle" identifiers out of a macro body with e.g. macros defined in the macro.
  So far, hygiene has been described as purely more strict---disallowing more programs than would be allowed with it, but in this case it actually does the opposite allowing programs that would be otherwise disallowed.

  All the above can be adapted for reasoning about privacy instead of scoping; the same rigid lexical principles apply.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
