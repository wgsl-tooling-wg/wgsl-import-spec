# Imports and Modules

## Motivation

When authoring bigger WGSL shaders, people typically **split code into reusable pieces**. Examples are reusable functions for lighting calculation or sharing a struct between two compute shaders.

Secondly, we want a module system that enables **importing shader from libraries**. Authors should be able to upload WGSL shaders to existing package managers, which other users can consume.

Finally, we want to enable **tooling** which natively understands import statements. Such tooling should exist for all WebGPU implementations and not be tied to a specific one.

### Example

The import in this example is derived from shaders in [the Bevy game engine](https://github.com/bevyengine/bevy/blob/9db9f9be2cfed58b39c7393e0cf9f50cc2321310/assets/shaders/show_prepass.wesl#L1-L3).

```wesl
import bevy_pbr::render::{
  mesh_view_bindings::globals, 
  forward_io::VertexOutput
};

import package::utils;

@fragment
fn fragment(mesh: VertexOutput) -> @location(0) vec43 {
  let color = utils::gamma(mesh.color);
  return color * globals.ambient_light;
}
```

# Guide-level explanation
The `import` statement extension brings item or module names into scope.

```wesl
// Importing a single function using a relative path
import super::lighting::pbr;

// Importing multiple items
import my::geom::sphere::{ draw, default_radius as foobar };

// Imports a module name. Use it with `bevy_ui::name`
import bevy_ui;

// Import all items from another module
import bevy::prelude::*;
import wgsl_test::expect::*;
```

These can then be used anywhere in the source code.

```wesl
fn main() {
    bevy_ui::quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project, typically installed by a package manager.

Recursive import definitions are also supported, which leads to shorter import statements.

```wesl
import bevy_pbr::{
  forward_io::VertexOutput,
  pbr_types::{PbrInput, pbr_input_new},
  pbr_bindings
};
```

*Import paths* map directly to modules and their items. Each module is stored
in a single source, typically a WESL or WGSL file: so
`bevy_pbr::pbr_types::PbrInput` names the item `PbrInput` declared in the
source file `pbr_types.wesl` in the `bevy_pbr` package.

# Reference-level explanation
A WESL program is composed of a tree of WESL modules.

Imports must appear at the beginning of a WESL file. They can bind the name of a module or of an individual "importable item" (see [GLOSSARY](GLOSSARY.md)).

### Grammar

An import statement is parsed with the following grammar, with spaces and comments allowed between tokens:

```ebnf
translation_unit:
| import_statement* global_directive* global_decl* 

import_statement:  
| attribute* 'public'? 'import' import_relative? (import_collection | import_path_or_item) ';'

import_relative:
| 'package' '::' | 'super' '::' ('super' '::')*

import_path_or_item:
| ident '::' (import_collection | import_path_or_item) 
| ident ('as' ident)?
| '*'

import_collection:
| '{' (import_path_or_item) (',' (import_path_or_item))* ','? '}'
```

Where `translation_unit` and `ident` are defined in the WGSL grammar.
`ident`s must not be current WGSL keywords. `ident`s also must not be
current WESL keywords: `as`, `import`, `package`, `public`, `self`, or `super`.
Reserved words that are not current keywords are allowed, but not recommended.
Lint tools may optionally warn when reserved words are used. 

Attributes may precede an import statement, notably `@if` for
[conditional translation](ConditionalTranslation.md) and `@diagnostic` for
[suppressible diagnostics](#suppressible-diagnostics).

An item import imports a single item. The item can be renamed with the `as` keyword.

An import collection imports multiple items, and allows for nested imports.

A wildcard import imports every top-level item visible to the importer: the
module's own visible declarations and its `public import` re-exports. Submodule
names and submodule contents are not imported. A wildcard must follow a module
path; a bare `import *;` is an error. A wildcard may also appear as a member of
an import collection, applying to the module path before the braces (see
[Import bindings](#import-bindings)).

The optional `public` prefix also re-exports the imported names under the importing module's path; see
[Visibility.md](Visibility.md).
After collection flattening, each `public import` must resolve to a single item.
The forms `public import path::*` and `public import some_module;` are not yet
assigned a meaning and are reserved.


### Import bindings

An import statement's recursive structure is first flattened, turning every `import_collection` into multiple separate imports.
For instance, `import a::{b, c::{d, e as f}};` would be turned into

```wesl
import a::b;
import a::c::d;
import a::c::e as f;
```

A wildcard may appear as a member of an import collection:
`import foo::{a::b, *};` becomes

```wesl
import foo::a::b;
import foo::*;
```

The wildcard imports only `foo`'s own top-level items: the sibling
branch reaching into submodule `foo::a` doesn't widen it.

Each flattened import statement binds one name in the importing module: the
last segment of its *import path*, or its `as` alias. The binding is a
shorthand for the import path; the import path itself need not name a module or
declaration. Each *reference*, a use of a name in code
(such as in a function call, type, attribute argument, or other expression),
determines a declaration path: a bound name expands to its import path, and
any `::` segments written after it extend the path
(see [Inline Usage](#inline-usage)).

How a bound name is used determines whether it refers to a declaration or a
module:

```wesl
import bevy_pbr::forward_io;

fn main() {
    var out: forward_io::VertexOutput;  // forward_io used as a module:
                                        //   the file bevy_pbr/forward_io.wesl
    let x = forward_io();               // forward_io used as a declaration:
                                        //   fn forward_io declared in bevy_pbr/package.wesl
}
```

A declaration and a module may share the same name: a bare `forward_io`
refers to the declaration, while `forward_io::VertexOutput` reaches into the
module.

### Resolving a declaration path

A *declaration path* is a fully qualified path whose final segment names a
declared item. A declaration path always begins with `package` or the name of
a known package; references written with `super` or bound names expand to
this form before resolution (see [Import bindings](#import-bindings)).
The segments before the final segment are the *module path*,
naming the module containing the declaration. In
`bevy_pbr::forward_io::VertexOutput`, `VertexOutput` is declared in the
module named by `bevy_pbr::forward_io`. Each module path names exactly one
*module source*: the stored text of a module, typically a file.

The first segment anchors the path:

* `package` anchors the path at the current package.
* Any other first segment must name a known package, and anchors the path at
  that package. Tools find the known packages in the
  [`wesl.toml`](WeslToml.md) file or through the host package manager's
  dependencies.

In a written path, a `super` prefix refers to the parent of the current
module: each `super` removes the last segment of the current module's path,
and it is an error to remove beyond the package root. The result is a
declaration path anchored at `package`.

A package provides a mapping from module paths to module sources; the
semantics of the module path segments beyond the first are specific to the
package's storage. Only the module path maps to storage; the final segment of
a declaration path names a declaration inside the module source, never a file.
The declaration may be declared in the module directly, or re-exported in the
module via `public import` (see [Visibility.md](Visibility.md)).
How a package maps module paths to sources depends on how it is stored (see
[Filesystem Resolution](#filesystem-resolution) and
[Non-Filesystem Resolution](#non-filesystem-resolution)).

A module path may consist of just `package`, or of just a bare package name.
Such a path refers to the optional *package root module*
(see [The package root module](#the-package-root-module)).

Referencing a declaration path that fails to resolve is an error. An import
statement whose bound name is never referenced is allowed, even if
referencing it would be an error; tools may warn about unused or unresolvable
imports. Import statements and references removed by
[conditional translation](ConditionalTranslation.md) are also allowed, even
if they would be errors under other conditions; tools may warn about these
too.

For a wildcard import, the entire path before the `*` is a module path; the
wildcard brings the module's top-level items into scope.

Resolution finds the declaration; whether the referencing module may then use
it is governed by [visibility](Visibility.md).

> [!NOTE]
> Tools can enumerate the potential resolutions an import statement
> enables by analyzing the source tree paths and the declarations in each
> module source, for example to suggest editor auto-completions.

### Examples

```wesl
import bevy_pbr::forward_io::VertexOutput;

@fragment
fn fragMain(v: VertexOutput) -> @location(0) vec4f { /* ... */ }
```

The reference `VertexOutput` in `v: VertexOutput` determines the declaration
path `bevy_pbr::forward_io::VertexOutput`. The module path is
`bevy_pbr::forward_io`, and the final segment refers to a declaration named
`VertexOutput` in that module (a struct, not shown). `bevy_pbr` is a package,
so the module source is the file `forward_io.wesl` in the `bevy_pbr`
package's root directory, or `forward_io.wgsl` if there is no `.wesl` file.
No other file is consulted; if both files are missing, the reference is an
error.

```wesl
// lighting/pbr.wesl
import super::shadowmapping;

fn shade() {
    let s = shadowmapping::pcf();
}
```

The current module is stored at `lighting/pbr.wesl`, with module path
`package::lighting::pbr`. `super` removes the last segment, giving
`package::lighting`. The import binds `shadowmapping` as a shorthand for
`package::lighting::shadowmapping`. The reference `shadowmapping::pcf()`
determines the declaration path `package::lighting::shadowmapping::pcf`,
referring to a function `pcf` declared in `lighting/shadowmapping.wesl` (not
shown).

## The package root module

The *package root module* is the optional module that the bare `package`
module path maps to. In filesystem resolution, it corresponds to a special file
named `package.wesl` in the package root directory.

A declaration `foo` in the package root module is addressable from outside the
package as `my_pkg::foo`, and from within the package as `package::foo`.

A `package` module is only allowed at the top level; tools should warn about a
module named `package` anywhere else.

## Filesystem Resolution

On a filesystem, a module path maps directly to a single file, following
[Resolving a declaration path](#resolving-a-declaration-path).

The resolution starts with a package root directory, typically in a
[`wesl.toml`](WeslToml.md#root-field) file, user-provided through the linker's API,
or defaulted by the linker.

> [!NOTE]
> Linkers may apply heuristics to find the default package root directory, such as
> looking for a `shaders` directory at the project root, or using the parent directory
> of the main module.

The package root module path is mapped to a file named `package.wesl` or `package.wgsl`
in the package root directory.

Other module paths are mapped as follows: each intermediate segment after `package`s
 names a subdirectory from the package root directory, and the last segment names
a `.wesl` or `.wgsl` file within the subdirectory.

The `.wesl` extension takes priority over `.wgsl` in case both files are present.
The extension has no other impact in the translation process.
A resolution fails if neither `.wesl` nor `.wgsl` file exists for a given module path.

**Example**:

Suppose the following project structure, and the package root directory set to `shaders/`:

```
src/
shaders/
  util/
    noise.wesl
  package.wesl
  main.wesl
  util.wgsl
  data.txt
package.json
```

The filesystem resolution maps:

* `package` to `shaders/package.wesl`
* `package::main` to `shaders/main.wesl`
* `package::util` to `shaders/util.wgsl`
* `package::util::noise` to `shaders/util/noise.wesl`
* `package::data` is an invalid module path, because there is no file named
  `data.wesl` or `data.wgsl` in `shaders/`.

### Reserved file names

Due to filesystem limitations, it can happen that WESL idents are invalid file or folder names.
Notable examples are `CON, PRN, AUX, NUL, COM1 - COM9, LPT1 - LPT9` on Windows, and Windows being case-insensitive.
We do not take these restrictions into account, instead we recommend avoiding these special names.

## Non-Filesystem Resolution

WESL resolution doesn't require a filesystem. It only requires a mapping from
module paths to module sources: a bundled package can store module sources in a
dictionary keyed by module path, and a package served over the web can map each
module path to a URL.

## Inline Usage
The syntax can also be used inline. To do so, we extend
the [WGSL grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent) as follows.

We introduce

```ebnf
full_ident:
| import_relative? ident ('::' ident)*
```

and then replace `ident` with `full_ident` in the following places:

```ebnf
core_lhs_expression:
| full_ident
| ...

for_init: 
| full_ident func_call_statement.post.ident
| ...

for_update:
| full_ident func_call_statement.post.ident
| ...

global_decl: 
| 'alias' ident '=' full_ident ...
| ...

primary_expression: 
| full_ident template_elaborated_ident.post.ident
| full_ident template_elaborated_ident.post.ident argument_expression_list
| ...

statement: 
| ...
| full_ident template_elaborated_ident.post.ident argument_expression_list ';'
| ...

type_specifier:
| full_ident ...
```

In an inline declaration path, the first segment may also be a name bound by
an import, which expands to its import path. Otherwise, the first segment
anchors the path as in
[Resolving a declaration path](#resolving-a-declaration-path): `package`,
`super`, or a known package name.

### Examples

```wesl
import foo::bar;
fn main() {
    let a = bar::baz; // bar expands to foo::bar, from the import above
    let b = bevy::main(); // Uses the known bevy package
}
```

## Cyclic Imports
Cyclic imports are allowed.

However, the following is still illegal.

```wesl
// foo.wesl
import package::bar::b;
const a = b + 1;
```

```wesl
// bar.wesl
import package::foo::a;
const b = a + 1;
```

(a depends on b which depends on a)

Basic linker implementations do not need to check for this. Generating broken code and letting the underlying shader compiler throw an error is fine.

## Wildcard imports

Wildcard imports bring every top-level item visible to the importer into the
importing module's scope (see
[Visibility between modules](Visibility.md#visibility-between-modules)).

Users can wildcard import:

- from any other module in the current package.
- from any external library module where the library author has added a
  `@!wildcardable` annotation.

Wildcard importing brings some stability risk when the imported module adds to
its API. The newly introduced names may conflict with other names in the
importer's namespace (from local definitions and other imports), leading to
compiler warnings or errors. To help mitigate this risk, WESL provides a
`@!wildcardable` annotation that library authors can place on modules that are
designed for wildcard importing.

Advanced users who wish to wildcard import from external modules not marked as
`@!wildcardable` can do so by suppressing `unsupported_wildcard` (see
[Suppressible diagnostics](#suppressible-diagnostics)).

```wesl
// wildcard import from a @!wildcardable external module
import bevy::prelude::*;
import wgsl_test::expect::*;

// wildcard import from within the current package
import package::utils::*;
import super::fun::*;
```

### `@!wildcardable` annotation

Library authors mark modules they intend for library consumers to wildcard
import with the `@!wildcardable;` module attribute (see [Grammar](#grammar)):

```wesl
// math.wesl  (in a library)
@!wildcardable;

public fn dot2(a: vec2f, b: vec2f) -> f32 { return a.x*b.x + a.y*b.y; }
public fn cross2(a: vec2f, b: vec2f) -> f32 { return a.x*b.y - a.y*b.x; }
```

### Recommendations for `@!wildcardable` modules

Because every `public` item in a `@!wildcardable` module is a potential
collision in importer code, library authors should curate these items carefully.

**Add public items hesitantly.** Adding a `public` item to a `@!wildcardable`
module is a semver minor change but can break users who have local declarations
or import other `@!wildcardable` modules.

- **Bundle** additions into a major release when one is upcoming.
- **Document** additions clearly in changelogs so downstream users debugging
  unexpected name resolution can trace them.

**Compose with re-exports.** See [Re-exports](Visibility.md#re-exports) to collect
items from other modules into a single `@!wildcardable` module for user
convenience.

**Avoid generic names.** Prefer domain-specific names. Common names like
`Buffer`, `Config`, `Result`, `Vec`, etc. are more likely to collide with user
applications.

**Don't shadow WGSL builtins.** Names like `vec3`, `clamp`, `inverseSqrt` have
expected semantics that should not be implicitly overridden with wildcards.
Similarly, avoid experimental Naga/Dawn/Safari builtins.
- The `builtin_shadow` diagnostic flags this (see
  [Suppressible diagnostics](#suppressible-diagnostics)).
- If a future WGSL update adds a conflicting builtin name, plan to update the
  `@!wildcardable` module to rename the conflicting item.

### Library-to-library wildcard imports

Libraries that wildcard import from other libraries raise special concerns. If a
user's package manager chooses a newer version of the imported-from library,
the user may see a conflict in library code they don't expect to modify.

Cross-package wildcard imports in library code
trigger the `cross_package_wildcard` diagnostic, a suppressible error. Library
authors can suppress the diagnostic with
`@diagnostic(off, cross_package_wildcard)` on the import statement, e.g. when
wildcard importing from external packages they control.

**Optional: publish-time wildcard expansion.** Library publishing tools may
also offer to expand wildcards to named imports in the published version of a
module, snapshotting the names at publish time:

```wesl
// source
import bevy::prelude::*;
```

```wesl
// published artifact
import bevy::prelude::{Color, Mesh, Transform, /* snapshot at publish time */};
```

Expansion pins the imported names so that a newer version of the imported-from
library can't introduce conflicts into already-published code.

## Import errors and warnings

The table below summarizes import errors and warnings. Resolution failures
and genuine collisions cannot be suppressed; other diagnostics are
suppressible via `@diagnostic`.

| Situation | Behavior |
| --- | --- |
| Reference fails to resolve (missing module source, or no such declaration) | Error |
| Local declaration conflicts with named import | Error |
| Named import conflicts with named import | Error |
| Wildcard imports provide different declarations for the same referenced name | Error |
| Wildcard import from a non-`@!wildcardable` external module | Error (`unsupported_wildcard`) on the import; suppressible |
| Cross-package wildcard import in library code | Error (`cross_package_wildcard`) on the import; suppressible |
| Local declaration or named import shadows a wildcard-imported name | Warning (`wildcard_shadow`) on the shadowing declaration or import; suppressible |
| `@!wildcardable` module exports an item shadowing a WGSL builtin | Warning (`builtin_shadow`) on the shadowing declaration; suppressible |

When multiple wildcard imports are in scope, the same name may be exported by
more than one module. If every wildcard resolves the name to the same
declaration, references are unambiguous and no error occurs. A name that could
refer to two different declarations is a dormant conflict: an error occurs
only where the name is referenced:

```wesl
import package::foo::*; // exports clashing_zap
import package::bar::*; // exports a different clashing_zap

fn main() {
    let x = clashing_zap(); // error: ambiguous between the two clashing_zap declarations
}
```

The fix is to disambiguate with a named import (`import package::foo::clashing_zap;`,
or `import package::foo::{clashing_zap, *};` to keep the wildcard) or an
[inline declaration path](#inline-usage) (`package::foo::clashing_zap()`).

### Suppressible diagnostics

Each diagnostic below can be suppressed at the site indicated with a
`@diagnostic` attribute, or module-wide with WGSL's
[global diagnostic directive](https://www.w3.org/TR/WGSL/#global-diagnostic-directive),
e.g. `diagnostic(off, wildcard_shadow);`.

- **`unsupported_wildcard`** fires on a wildcard import from an external module
  that doesn't support wildcard import (not marked `@!wildcardable`). Suppress
  with `@diagnostic(off, unsupported_wildcard)` on the import statement to
  accept the upgrade risk that future versions of the imported module may add
  conflicting names. The suppression has no effect on an import from a
  `@!wildcardable` module, where the diagnostic never fires.

- **`wildcard_shadow`** is a warning that fires on a local declaration or named
  import that shadows a name brought in by a wildcard import. The shadowing is
  allowed: the local declaration or named import wins by precedence
  (see [Scope precedence](#scope-precedence)). Suppressing the warning with
  `@diagnostic(off, wildcard_shadow)` on the shadowing declaration or import
  statement changes only the reporting, not the resolution.

- **`cross_package_wildcard`** is a suppressible error that fires on a
  cross-package wildcard import in library code (see
  [Library-to-library wildcard imports](#library-to-library-wildcard-imports)).
  Suppress with `@diagnostic(off, cross_package_wildcard)` on the import
  statement.

- **`builtin_shadow`** is a warning that fires in a `@!wildcardable` module,
  on a top-level declaration whose name shadows a WGSL builtin such as `vec3`
  or `clamp`, whether or not any module wildcard imports it. Suppress with
  `@diagnostic(off, builtin_shadow)` on the offending declaration if the
  override is intentional.

## Scope precedence

When a name could resolve to items at multiple precedence levels, the
highest-precedence one wins. The table in
[Import errors and warnings](#import-errors-and-warnings) lists the overlaps
that produce a warning or an error; other overlaps resolve silently.

1. user declarations and named imports (non-wildcard)
2. wildcard-imported names
3. predeclared items (WGSL builtins)

User declarations and named imports share the top precedence level: a conflict
between them is an error, so at most one candidate can exist at that level.

Predeclared items rank lowest so that future WGSL spec revisions can add new
builtins without breaking existing shaders: any name already bound at a higher
level continues to resolve as before. Wildcard-imported names rank below user
declarations and named imports for the analogous reason: additions to a
`@!wildcardable` module won't silently change resolution at call sites that
already have a local or named binding for the same name.

### Module and package names form a separate namespace from declarations

In a reference, a path segment followed by `::` names a module or a package,
and a bare name refers to a declaration. So a declaration may share its name
with a package without ambiguity:

```wesl
import light::foo;

const bar: light::foo = 0; // `light::` unambiguously names the package

fn light() {}              // no conflict with the package name
```

Wildcard imports preserve this separation: they bring in only the imported
module's top-level items, never module or package names.

Within the namespace, an import binding shadows a package with the same name:
a first segment refers to a bound name when one is in scope, and otherwise to
a package (see [Inline Usage](#inline-usage)). Tools may warn about the
shadowing.

## Side-effects and `const_assert`
Generally, WGSL elements are included if they are recursively used from the main module ([statically accessed](https://www.w3.org/TR/WGSL/#statically-accessed)).
An import statement by itself doesn't have any side effects. It does not bring in `const_assert`s.

`const_assert` statements are also included if they are in the same module or namespace as a used element.
This only refers to the exact module that an element is in, and not any of the parent modules.
`let a = bevy_pbr::lighting::shadows::SHADOW_DEPTH;` would bring in the `const_assert`s of the `shadows` module.

Example:

```wesl
// main.wesl:
import package::foo::bar;
fn main() { bar(); }

// foo.wesl:
import package::zig::zag;
const_assert(1 > 0); // included in link because bar is used
fn bar() { }
fn miz() { zag() }

// zig.wesl:
const_assert(2 < 0); // not included in link
fn zag() { }
```

Example

```wesl
import package::foo::bar;

// Only the package::foo::bar::baz module would bring in its const assertions
const a: u32 = bar::baz::hello; 
```

`const_assert`s inside functions are treated specially! They can get eliminated during dead-code elimination, which is an observable side-effect.
[WESL deviates from WGSL here](https://github.com/webgpu-tools/wesl-spec/issues/93).

## Dead Code Elimination
Linkers may choose to do dead code elimination, but it is a non-observable implementation detail.

`const_assert` statements inside of functions need special treatment, see relevant section.

# Drawbacks
Are there reasons as to why we should not do this?

* This introduces yet another importing syntax that developers have to learn, instead of using a standard syntax.
* To implement the name mangling, one has to parse WGSL code! This is not trivial, and requires a partial WGSL parser.
* Paths in import statements must consist of valid WGSL identifiers, which can be limiting. This limitation could be lifted by allowing arbitrary strings in import paths, but would make the implementation more complex.

# Rationale and alternatives

## TypeScript-like imports
The Bevy team, with a large shader codebase, had a few wishes

* A short syntax
* Inline usage

## Rust-like imports
To fully copy Rust's importing syntax, one needs something akin to a `mod` statement.
The rules have carefully been architected to imitate the Rust style, while not requiring an explicit `mod` statement.

In Rust, `use foo::bar;` could either map to "import an item called `bar` from `foo.rs`" or it could map to "import the module `foo/bar.rs`". Rust uses the explicit `mod` statement to disambiguate. We instead decide at each reference: a bare `bar` is the item, and `bar::baz` reaches into the module.

# Future possibilities

## Namespaces
We hope that namespaces will be added to WGSL itself. Then, the importing mechanism can be extended to fully support namespaces, for example by treating each file as introducing its own namespace.

## Source maps
We encourage tooling authors to also implement source maps when implementing imports. This aids

* Error Reporting. When Naga or Tint report an error in the generated WGSL code, we want to map the error location back to the WESL code.
* Debugging. Eventually we hope to have a full toolchain of WESL to WGSL to SPIR-V, with source maps at each step. In the end, it should be possible for RenderDoc to show the original WESL code.

## Scoped imports
Allow imports that are only active within one function?
