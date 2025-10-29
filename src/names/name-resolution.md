r[names.resolution]
# Name resolution

r[names.resolution.intro]

_Name resolution_ is the process of tying paths and other identifiers to the
declarations of those entities. Names are segregated into different
[namespaces], allowing entities in different namespaces to share the same name
without conflict. Each name is valid within a [scope], or a region of source
text where that name may be referenced. Access to certain names may be
restricted based on their [visibility].

Name resolution is split into three stages throughout the compilation process. The first stage, Expansion-time resolution, resolves all [use declarations] and [macro invocations]. The second stage, Primary resolution, resolves all names that have not yet been resolved that do not depend on type information to resolve. The last stage, Type-relative resolution, resolves the remaining names once type information is available.

> Note
>
> * Expansion-time resolution is also known as "Early Resolution"
> * Primary resolution is also known as "Late Resolution"

r[names.resolution.expansion]
## Expansion-time name resolution

r[names.resolution.expansion.intro]

Expansion-time name resolution is the stage of name resolution necessary to complete macro expansion and fully generate a crate's AST. This stage requires the resolution of macro invocations and use declarations. Resolving use declarations is required to resolve [path-based scope] macro invocations. Resolving macro invocations is required in order to expand them.

After expansion-time name resolution, the AST must not contain any unexpanded macro invocations. Every macro invocation resolves to a valid definition that exists in the final AST (or an external crate). The resolution of imports must be *stable*. After expansion, imports in the fully expanded AST must resolve to the same definition, regardless of the order in which macros are expanded. Once the crate has been fully expanded all speculative import resolutions are validated to ensure that no new ambiguities were introduced by macro expansion.

> Note
>
> Due to the iterative nature of macro expansion, this causes so called time
> traveling ambiguities, such as when a glob import introduces an item that is
> ambiguous with its own base path.
>
> ```rust,compile_fail,E0659
> macro_rules! m {
>     () => { mod bar {} }
> }
>
> mod bar {
>     pub(crate) use m;
> }
>
> fn f() {
>     // * initially speculatively resolve bar to the module in the crate root
>     // * expansion of m introduces a second bar module inside the body of f
>     // * expansion-time resolution finalizes resolutions by re-resolving all
>     //   imports and macro invocations, sees the introduced ambiguity
>     //   and reports it as an error
>     bar::m!(); // ERROR: `bar` is ambiguous
> }
> ```

r[names.resolution.expansion.imports]

All use declarations are fully resolved during this stage of resolution. Type-relative paths cannot be resolved at this stage of compilation and will produce an error.

```rust,compile_fail,E0432
mod my_mod {
    pub const Const: () = ();

    pub enum MyEnum {
        MyVariant
    }

    impl MyEnum {
        pub const Const: () = ();
    }

    pub type TypeAlias = MyEnum;
}

fn foo() {
    // imports resolved at expansion-time
    use my_mod::MyEnum; // OK
    use my_mod::MyEnum::MyVariant; // OK
    use my_mod::TypeAlias; // OK
    use my_mod::TypeAlias::MyVariant; // Doesn't work
    use my_mod::MyEnum::Const; // Doesn't work
    use my_mod::Const; // OK

    // expressions resolved during type-relative resolution
    let _ = my_mod::TypeAlias::MyVariant; // OK
    let _ = my_mod::MyEnum::Const; // OK
}
```

r[names.resolution.expansion.imports.shadowing]

The following is a list of situations where shadowing of use declarations is permitted:

* [use glob shadowing]
* [macro textual scope shadowing]

r[names.resolution.expansion.imports.ambiguity]
## Ambiguities

r[names.resolution.expansion.imports.ambiguity.intro]
Some situations are an error when there is an ambiguity as to which macro definition, use declaration, or module an import or macro invocation's name refers to. This happens when there are two name candidates that do not resolve to the same entity where neither candidate is [permitted] to shadow the other.

r[names.resolution.expansion.imports.ambiguity.globvsglob]
Names may not be resolved through ambiguous glob imports. Glob imports are allowed to import conflicting names in the same namespace as long as the name is not used. Names with conflicting candidates from ambiguous glob imports may still be shadowed by non glob imports and used without producing an error. The errors occur at time of use, not time of import.

For example:

```rust
mod foo {
    pub struct Qux;
}

mod bar {
    pub struct Qux;
}

use foo::*;
use bar::*; //~ OK, no name conflict.

fn ambiguous_use() {
    // This would be an error, due to the ambiguity.
    //let x = Qux;
}

fn ambiguous_shadow() {
    // This is permitted, since resolution is not through the ambiguous globs
    struct Qux;
    let x = Qux;
}
```

Multiple glob imports are allowed to import the same name, and that name is allowed to be used, if the imports are of the same item (following re-exports). The visibility of the name is the maximum visibility of the imports. For example:

```rust
mod foo {
    pub struct Qux;
}

mod bar {
    pub use super::foo::Qux;
}

// These both import the same `Qux`. The visibility of `Qux`
// is `pub` because that is the maximum visibility between
// these two `use` declarations.
pub use bar::*;
use foo::*;

fn main() {
    let _: Qux = Qux;
}
```

r[names.resolution.expansion.imports.ambiguity.globvsouter]
it is an error to shadow an outer name binding with a glob import.

```rust,compile_fail,E0659
mod bar {
    pub mod foo {
        pub struct Name;
    }
}

mod baz {
    pub mod foo {
        pub struct Name;
    }
}

pub mod foo {
    pub struct Name;
}

pub fn qux() {
    use bar::*;
    use foo::Name; // ERROR: `foo` is ambiguous
}
```

```rust,compile_fail,E0659
pub mod bar {
    #[macro_export]
    macro_rules! m {
        () => {};
    }

    macro_rules! m2 {
        () => {};
    }
    pub(crate) use m2 as m;
}

pub fn qux() {
    use bar::*;
    m!(); // ERROR: `m` is ambiguous
}
```

> **NOTE** These ambiguity errors are specific to imports, even though they are
> only observed when those imports are used, having multiple candidates
> available for a given name during later stages of resolution is not
> considered an error, so long as none of the imports themselves are ambiguous,
> there will always be a single unambiguous closest resolution during later
> stages.
>
> ```rust
> mod bar {
>     pub struct Name;
> }
>
> mod baz {
>     pub struct Name;
> }
>
> use baz::Name;
>
> pub fn foo() {
>     use bar::*;
>     Name; // resolves to bar::Name
> }
> ```

r[names.resolution.expansion.imports.ambiguity.moreexpandedvsouter]
It is an error for name bindings from macro expansions to shadow name bindings from outside of those expansions.

```rust,compile_fail,E0659
macro_rules! name {
    () => {}
}

macro_rules! define_name {
    () => {
        macro_rules! name {
            () => {}
        }
    }
}

fn foo() {
    define_name!();
    name!(); // ERROR: `name` is ambiguous
}
```

r[names.resolution.expansion.imports.ambiguity.pathvstextualmacro]
Path-based scope bindings for macros may not shadow textual scope bindings to macros. For bindings from [use declarations], this applies regardless of their [sub-namespace].

```rust,compile_fail,E0659
#[macro_export]
macro_rules! m2 {
    () => {}
}
macro_rules! m {
    () => {}
}
pub fn foo() {
    m!(); // ERROR: `m` is ambiguous
    use crate::m2 as m; // in scope for entire function body
}
```

r[names.resolution.expansion.imports.ambiguity.builtin-attr]
It is an error to use a user defined attribute or derive macro with the same name as a builtin attribute (e.g. inline).

<!-- ignore: test doesn't support proc-macro -->
```rust,ignore
// myinline/src/lib.rs
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn inline(_attr: TokenStream, item: TokenStream) -> TokenStream {
    item
}
```

<!-- ignore: requires external crates -->
```rust,ignore
// src/lib.rs
use myinline::inline;
use myinline::inline as myinline;

#[myinline::inline]
pub fn foo() {}

#[crate::inline]
pub fn bar() {}

#[myinline]
pub fn baz() {}

#[inline] // ERROR: `inline` is ambiguous
pub fn qux() {}
```

r[names.resolution.expansion.imports.ambiguity.derivehelper]

Helper attributes may not be used before the macro that introduces them.

> [!NOTE]
> rustc currently allows derive helpers to be used before their attribute macro
> introduces them into scope so long as they do not shadow any other attributes
> or derive helpers that are otherwise correctly in scope. This behavior
> deprecated and slated for removal.
> <!-- ignore: requires external crates -->
> ```rust,ignore
> #[helper] // deprecated, hard error in the future
> #[derive(WithHelperAttr)]
> struct Struct {
>     field: (),
> }
> ```
>
> For more details, see [Rust issue #79202](https://github.com/rust-lang/rust/issues/79202).

r[names.resolution.expansion.macros]

Macros are resolved by iterating through the available scopes until a candidate
is found. Macros are split into two sub-namespaces, one for bang macros, and
the other for attributes and derives. Resolution candidates from the incorrect
sub-namespace are ignored. The available scopes are visited in the following order.

* derive helpers
    * not visited when resolving derive macros in the parent scope (starting scope)
* derive helpers compat
* textual scope macros
* path-based scope macros
* macrouseprelude
    * not visited in 2018 and later when `#[no_implicit_prelude]` is present
* stdlibprelude
* builtinattrs

r[names.resolution.expansion.macros.errors.reserved-names

the names cfg and cfg_attr are reserved in the macro attribute [sub-namespace].

r[names.resolution.late]

> [!NOTE]
> This is a placeholder for future expansion.

r[names.resolution.type-dependent]

> [!NOTE]
> This is a placeholder for future expansion.

[use glob shadowing]: ../items/use-declarations.md#r-items.use.glob.shadowing
[Macros]: ../macros.md
[use declarations]: ../items/use-declarations.md
[macro textual scope shadowing]: ../macros-by-example.md#r-macro.decl.scope.textual.shadow
[`let` bindings]: ../statements.md#let-statements
[item definitions]: ../items.md
[namespaces]: ../names/namespaces.md
[scope]: ../names/scopes.md
[visibility]: ../visibility-and-privacy.md
[permitted]: name-resolution.md#r-names.resolution.expansion.imports.shadowing
[macro invocations]: ../macros.md#macro-invocation
[path-based scope]: ../macros.md#r-macro.invocation.name-resolution
[sub-namespace]: ../names/namespaces.md#r-names.namespaces.sub-namespaces
