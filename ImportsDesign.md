# Imports Design

This document records design decisions behind WESL's import system. The
normative spec lives in [Imports.md](Imports.md).

## Why the `@!` module attribute form?

`@!wildcardable` is module-level metadata. WESL will likely want module-level
annotations for other module-scoped features, and libraries and users will want
a place to attach their own metadata to a whole module. A general-purpose syntax
for module metadata avoids inventing one ad-hoc form per feature.

`@!wildcardable` mirrors the item-level attribute convention (`@group`,
`@binding`, `@if`, `@diagnostic`, ...), but the `!` marks the attribute as
scoped to the whole module. Unlike an item attribute, it doesn't attach to a
following element, so a trailing `;` terminates it.

Module attributes sit below the imports so that they can use imported
names (for example a hypothetical `@!play_version(2);`).

See [`@!wildcardable` annotation](Imports.md#wildcardable-annotation) for the
normative spec.

## Aren't wildcards an anti-pattern?

Many language communities discourage wildcard imports. TypeScript, Go, Zig, and
Carbon disallow wildcards entirely or restrict them to narrow cases. Java and
Rust permit them syntactically but discourage broad use by convention; Rust's
`prelude` modules are one curated pattern in that style. The general concerns
are practical:

- **Traceability.** Direct imports make it obvious where a name comes from.
  Wildcards push that work onto the reader, the language server, or the
  compiler's name-resolution diagnostics.
- **API stability.** Adding a public item to a wildcard-imported module can
  conflict with downstream declarations or with other wildcard imports. Stacked
  wildcards across a dependency tree can create conflicts the end user neither
  caused nor can easily fix.

The WESL environment adds further concerns:

- **Cross-ecosystem publishing.** WESL libraries can be published into multiple
  host ecosystems (npm, crates, etc.), and the language's stability rules have
  to work for all of them. npm in particular treats minor/patch breakage as an
  upstream bug, so wildcard-driven conflicts on additive package updates would
  be read there as buggy packages, not as users accepting a WESL-specific
  tradeoff. The defaults can't be split per-ecosystem; even libraries that
  aren't actively cross-published inherit the same rules.
- **Mixed-language ownership.** In host applications, dependency updates are
  often routine maintenance handled by someone other than the shader author. A
  wildcard conflict can land on a teammate who did not cause it and may not be
  best positioned to fix shader-side breakage.
- **Shader test coverage.** Shader test coverage is often thinner than
  application-code coverage, and some failures are visual or runtime-dependent.
  Fewer tests and WGSL's comparatively small type system mean that wildcard
  conflicts are less likely to be caught at the moment a dependency is updated.
- **Single namespace.** WGSL has a single namespace for types and values, and no
  namespace construct or object-style surface to limit the scope of wildcarded
  names after import. There are fewer places for names to coexist harmlessly.

These concerns motivate guardrails for WESL wildcard defaults.

## Wildcards in WESL: when to allow, when to gate

WESL keeps wildcards available because some libraries are designed to feel
pervasive. Game engines, test frameworks, and math libraries expect a
domain-specific API where prefixing every call with `test::expect::` or similar
would obscure the shader rather than help it. Concise import syntax matters even
where an IDE can autocomplete: not every editor has a language server, and long
import blocks add noise regardless of how they were typed.

But the concerns in
[Aren't wildcards an anti-pattern?](#arent-wildcards-an-anti-pattern) still
apply, especially across package boundaries. WESL's defaults try to keep the
benefits while limiting the risk:

- **Not every public module suits wildcards.** Modules with a fast-growing API
  or with generic names (`Buffer`, `Result`) are fine to import by name but
  hazardous to wildcard.
- **Authors can signal which modules are curated for wildcards.** An explicit
  `@!wildcardable` marker lets library authors tell consumers (and tools) which
  modules they've designed for wildcard use. It also gives tooling a hook for
  lints around generic names, builtin shadowing, churn-prone additions, etc.
- **Defaults shape the ecosystem.** Red/yellow squiggles and linter messages
  teach safe wildcard practice to new and part-time shader authors more reliably
  than community blogs or documentation.
- **Advanced users are not blocked.** Within a package, wildcard imports are
  unrestricted; externally, wildcard-importing a non-`@!wildcardable` module is
  possible via
  [standard diagnostic controls](Imports.md#suppressible-diagnostics). The
  default tunes the path of least resistance, but doesn't block users who
  intentionally accept the risk.

## Alternatives


### Composing shader code as strings at runtime
One alternative is to compose shader code at runtime
by simply joining together strings with WGSL code, perhaps
with some string templating for flexibility.
This has the major downside of not being statically analyzable.
The IDE cannot provide autocompletion,
and a language server cannot check for errors.

A linker that understands imports also typically
composes shader strings, and can link at runtime.
But a linker uses its more sophisticated understanding of WGSL
to drive composition.
For example, a linker can identify imports
that are needed by other imports,
automating shader composition for users.

### Preprocessor `#include <lighting.wgsl>`
One alternative, which is common in the GLSL and C worlds, is an including mechanism which simply copy-pastes existing code. A major upside is that this is very simple to implement.

One drawback is that importing the same shader multiple times, which can also happen indirectly, does not work without other preprocessor features.

```c
// A.wgsl
#include <lighting.wgsl>
#include <math.wgsl>
```

```c
// lighting.wgsl
#include <math.wgsl>
```

would not work, since anything defined in `math.wgsl` would be imported twice. In C-land, this is solved by using *include guards*.

Another drawback is that using the same name twice is impossible. In C-land, this leads to pseudo-namespaces, where major libraries will prefix all of their functions with a few symbols. An example of this is the Vulkan API `vkBeginCommandBuffer` or `vkCmdDraw`.

A future drawback is that "privacy" or "visibility" becomes very difficult to implement. Everything that is imported is automatically public and easily accessible.
In C-land, the workaround is using header files. In other languages, such as Python, the convention ends up being "anything prefixed with an underscore `_` is private".

### Putting exports in comments
This would have the advantage of letting some existing WGSL tools ignore the new syntax. For example, a WGSL formatter would not need to know about imports, and could just format the code as usual.

### Using an alternative shader language
There are multiple higher level shading languages, such as [slang](https://github.com/shader-slang/slang) or [Rust-GPU](https://github.com/EmbarkStudios/rust-gpu) which support imports. They also support more features that WGSL currently does not offer. For complex projects, this can very much pay off.

The downside is using additional tooling, and dealing with an additional translation layer.
An additional translation layer could lock shader authors out of certain WGSL features.

Also, higher level GPU languages are typically processed at build time,
which precludes using language features to adapt to runtime conditions
like GPU characteristics or user settings.