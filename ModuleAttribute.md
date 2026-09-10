# Module Attributes

## Motivation

Adding module level metadata enables better tools.
For example, one can imagine marking a module as deprecated or as containing test code.

### Example

Wildcards are a semver hazard, so opting into them is done explicitly. A module level attribute is used.

```wesl
// math.wesl  (in a library)
@!wildcardable;

public fn dot2(a: vec2f, b: vec2f) -> f32 { return a.x*b.x + a.y*b.y; }
public fn cross2(a: vec2f, b: vec2f) -> f32 { return a.x*b.y - a.y*b.x; }
```

# Guide-level explanation

WESL extends WGSL's `global_directive` rule with a *module attribute*: a `@!`-prefixed attribute that carries module-level metadata. It is used by `@!wildcardable` (see [Wildcard imports](Imports.md#wildcard-imports)) and is otherwise reserved for future use.

### Grammar

```ebnf
global_directive:
| ... // existing WGSL forms
| module_attribute_directive

module_attribute_directive:
| '@' '!' ident_pattern_token argument_expression_list? ';'
```

A module attribute is written like a WGSL `attribute` with a `!` immediately
after the `@`, and is terminated with `;`; `ident_pattern_token` and
`argument_expression_list` are the WGSL rules. Like other global directives,
module attributes appear after any imports and before any global declarations,
and apply to the module they appear in.

