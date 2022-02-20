# DotNetNamingRulesLangServer

On top of the basic `key = value` syntax of .editorconfig, the .NET compilers (C#, VB.NET and perhaps others) [can use .editorconfig files to enforce (or ignore) naming conventions for various code elements](https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/naming-rules).

Conceptually, this system has three kinds of entities, depending on whether the key starts with `dotnet_naming_style.`, `dotnet_naming_symbols.`, or `dotnet_naming_rule.`:

* Symbol groups define which symbols a convention should be applied to
* Naming styles define what the symbol should look like under a given convention,
* Naming rules specify that symbol group "x" should follow naming style "y"; otherwise report it (or ignore it) with the specified severity level.

Entities are defined by one or more properties in the following syntax:

```ini
<kind>.<entityName>.<propertyName> = <propertyValue>
```

where `<kind>` is one of the above values, `<entityName>` is the name of the entity, and `<propertyName>` and `<propertyValue>` are specific to each entity kind.

The language server will provide language capabilities for this added system for naming rules, as described below:

* [Completions](#completions)
* [Symbol renaming](#symbol-renaming)
* [Find all references](#find-all-references)
* [Hover](#hover)
* [Navigation](#navigation)
* [Diagnostics](#diagnostics)

Given the following sample:

```ini
[*.{cs,vb}]
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_symbols.private_static_fields.applicable_kinds = field
dotnet_naming_symbols.private_static_fields.applicable_accessibilities = private
dotnet_naming_symbols.private_static_fields.required_modifiers = static

dotnet_naming_rule.private_fields_underscored.symbols = private_fields
dotnet_naming_rule.private_fields_underscored.style = underscored
dotnet_naming_rule.private_fields_underscored.severity = error

dotnet_naming_rule.private_static_fields_none.symbols = private_static_fields
dotnet_naming_rule.private_static_fields_none.style = underscored
dotnet_naming_rule.private_static_fields_none.severity = none

dotnet_naming_rule.public_members_must_be_capitalized.symbols  = public_symbols

dotnet_naming_style.underscored.capitalization = pascal_case

[*.cs]
dotnet_naming_style.underscored.required_prefix = _
```

## Completions

| When typing this: | Intellisense: | Comments |
| --- | --- | ---|
| `dotnet_naming_symbols.` | `private_fields`<br/>`private_static_fields`<br/>`public_symbols` | The first two are used in actual symbol definitions.<br/>`public_symbols` is used as the value for a naming rule `symbols` property. |
| `dotnet_naming_rule.public_members_must_be_capitalized.symbols  =` | Same as above | |
| `dotnet_naming_symbols.private_fields.` | `applicable_kinds`<br/>`applicable_accessibilities`<br/>`required_modifiers` | The properties available for symbol groups |
| `dotnet_naming_symbols.private_fields.applicable_kinds =` | `abstract`<br/>`async`<br/>`const`<br/>`must_inherit`<br/>`readonly`<br/>`static`<br/>`shared` | The available values for this property |

## Symbol renaming

Invoking the Rename refactoring on an entity name should modify all uses of that name -- both the keys in property setters, and the values in the naming rule setters. For example, Rename refactoring the `underscored` in the value of the following line:

```
dotnet_naming_rule.private_fields_underscored.style = underscored
```

to `under_scored`, should result in the following other changes:

```
dotnet_naming_style.under_scored.capitalization = pascal_case
dotnet_naming_style.under_scored.required_prefix = _
dotnet_naming_rule.private_static_fields_none.style = under_scored
```

Question: should this affect only the current file, or .editorconfig files in the parent folders as well?

## Find all references

Invoking "Find all references" on a specific name should return all property setters with that kind+name -- from all glob patterns, but only the last one for any given glob pattern (because that's the resolved one) -- as well as all the places it's used in the value of a property setter.

## Hover

| Hover over | Displays | Notes |
| --- | --- | --- |
| `capitalization` | `Capitalization style for words within the symbol` | |
| `local` | `Symbols defined within a method` | |
| `underscored` | `.capitalization = pascal_case (*.{cs,vb})`<br/>`.required_prefix = _ (*.cs)` | 1. Needs to handle multiple glob patterns.<br/>2. The glob pattern should be visually different from the value |

## Navigation

"Go to definition" on a specific name should navigate to the first property setting with that kind+name. Alternatively, it should follow the same strategy VS does with C# partial classes -- it would automatically open the **Find all references** pane on that kind+name.

## Diagnostics

The last line in the sample should have a red squiggly underneath it, because there are no property setters matching the `public_symbols` symbol group.

Missing required properties should also be recognized as errors.

Warn on symbol groups or naming stles that are not used in any naming rule.
