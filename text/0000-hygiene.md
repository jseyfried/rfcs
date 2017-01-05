- Feature Name: hygiene
- Start Date:
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Hygiene for declarative macros 2.0.

# Motivation
[motivation]: #motivation

## Names in macro definitions behave reliably

More specifically, names in macro definitions behave independently of
 - where the macro is invoked (i.e. in which crate/module/scope)
 - what names are used in the macro invocation arguments,
   - except when matching against names in patterns.

### Reliably use items accessible from the macro definition

Today, the only way a macro author can reliably use an item from outside the macro definition
is to employ a `$crate::`-prefixed path.

This is cumbersome and doesn't allow macro authors reliably use
 - trait items (since the trait methods in scope are not reliable), or
 - anything not accessible from outside the crate.

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

Today, a macro author cannot reliably define and use an item;
it is always possible for a macro user to make the item to trigger a conflict error.

A reliable definition does not conflict with names from outside the macro definition
and has no effect on the resolution of names from outside the macro definition.

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

## Use macro arguments in more places

For example, consider
```rust
macro m($it:item, $t:ty) {
    mod foo {
        $it
    }
}

fn f() { ... } // If `fn f() { ... }` is a valid item,
m!(fn f() { ... }); // then this should compile.
```

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

What names and terminology work best for these concepts and why? 
How is this idea best presented—as a continuation of existing Rust patterns, or as a wholly new one?

Would the acceptance of this proposal change how Rust is taught to new users at any level? 
How should this feature be introduced and taught to existing Rust users?

What additions or changes to the Rust Reference, _The Rust Programming Language_, and/or _Rust by Example_ does it entail?

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
