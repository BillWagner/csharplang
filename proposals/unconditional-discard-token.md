# Unconditional discard token

Champion issue: TBD

## Summary
[summary]: #summary

Today the token `_` is either an *identifier* (a name for a local, parameter, field, type, constant, ...) or a *discard*, and which one it is depends on the syntactic context, on how many `_` appear in the same list, on whether the construct existed before discards were added, and on what simple-name lookup of `_` finds. This proposal makes `_` **always a discard-token**. It is never an identifier and never binds to a declaration. A program element intended to be named `_` must be written `@_`. That includes types and namespaces: `_` is neither a type nor a variable (see [Type `_`](#type-_)).

```csharp
int _ = Compute();          // today: a local named `_`;   proposed: error at the declaration (write @_ for a named local).
Use(_);                     // today: reads that local;    proposed: error ('_' is a discard; write @_ if you meant a name)
Func<int, int> f = _ => 0;  // today: a parameter named `_`; proposed: a discard parameter
int @_ = Compute();         // identifier named `_` in every version of the language, today and proposed
```

## Motivation
[motivation]: #motivation

`_` was an ordinary identifier long before C# 7.0 introduced discards. To avoid breaking existing programs, each place that later learned to accept a discard kept the old meaning whenever the old syntax was legal. The result is that the meaning of `_` is determined by *history* rather than by a rule a developer can state. Everything below was verified by compiling and running the code with the .NET SDK 11 preview compiler (see the companion `existing-tests.cs`, described in [Test programs](#test-programs)).

**The same token has different meanings in neighbouring constructs.**

| Code | Meaning of `_` today |
| --- | --- |
| `int _ = 1;` / `var _ = 1;` | A *named local*. `var _ = 1; var _ = 2;` is error CS0128. |
| `foreach (var _ in xs)`, `for (int _ = 0; ...)`, `using var _ = ...`, `catch (E _)` | A *named local*; a second one in the same or nested scope is CS0128/CS0136. |
| `M(int _)` (method, local function, constructor, record/primary-constructor parameter) | A *named parameter*. `M(int _, int _)` is CS0100. |
| `_ => _ + 1`, `(_, x) => _ + x` | A *named lambda parameter* (only one `_` in the list). |
| `(_, _) => 0`, `(int _, int _) => 0` | *Discard parameters* (two or more `_` in the same list). `(_, _) => _` is CS0103, or silently binds to an *outer* `_`. |
| `(_, @_) => ...` | One `_` plus one `@_`: `@_` is not counted, so both are named parameters with the same name: CS0100. |
| `out _` | A *discard* if nothing named `_` is in scope; otherwise it *assigns the in-scope variable* (CS1503 if its type does not match). |
| `out var _`, `out int _`, `o is int _`, `o is var _`, `var (_, x) = ...`, `(int _, int y) = ...` | *Always* a discard, even if a variable named `_` is in scope. |
| `_ = e;` | A discard if nothing named `_` is in scope, otherwise an *assignment*. If `_` is a method group the result is CS1656. |
| `(_, x) = (1, 2);` | A discard if nothing is in scope, otherwise an assignment to the in-scope `_`, or an error if the type of `_` is incompatible. |
| `foreach ((_, x) in e)` | Error CS8186 whether or not `_` is in scope. |
| `e switch { ... _ => ... }` | *Always* the discard pattern, even when a local, field **or constant** named `_` is in scope. |
| `switch (e) { case _: }` | Error CS0103 if nothing named `_` exists; the *constant pattern* (warning CS8512) if a constant `_` is in scope; the *type pattern* if a type `_` is in scope. |
| `e is _` | Error CS0246 if nothing named `_` exists or if `_` is a constant/field (the right operand is bound as a type); CS0118 if `_` is a local; the *is-type test* (warning CS8513) if a type `_` is in scope. |
| `(a, b) is (_, _)`, `s is { Length: _ }` | *Always* discard subpatterns, even if a constant or type `_` is in scope. |
| `o is _ x`, `o switch { _ x => 1 }`, `t is (_ a, _ b)` with a type named `_` | A *parse error*: the parser never treats the token `_` as the start of a type in a pattern (the type must be written `@_`). |
| `nameof(_)`, `$"{_}"` | The name of / value of a named `_` if one is in scope; CS0103 otherwise. |

**The specified rules do not match the behavior.** The current standard text (§11.2.7 *Discard pattern*) says that an accessible constant named `_` turns a `_` in *any* pattern context into a constant pattern, and that a type named `_` is permitted as the type of a declaration pattern (`_ x`). There is an open issue to fix, but the above table shows how complicated that is as we move toward the specification for C# 14.

**Developers get surprising results from perfectly reasonable code.**

```csharp
var _ = Task.Run(Work);          // reads as a discard, is a local named '_'
var _ = Task.Run(OtherWork);     // CS0128 - "already defined"

int.TryParse(s, out _);          // discard...
string _ = "label";              // ...until someone adds an unrelated local
int.TryParse(s, out _);          // now CS1503, far from the declaration that changed the meaning

Func<int, int, int> f = (_, _) => _;   // CS0103
Func<int, int, int> g = (_, x) => _;   // fine: '_' is the first parameter
```

**The compiler and tools pay for it too.** The decision is spread across the parser (`IsTrueIdentifier`, and pattern/designation parsing in `LanguageParser_Patterns.cs` and `LanguageParser.cs` that special-cases the contextual `UnderscoreToken`), the binder (`FallBackOnDiscard` for deconstruction and `out`, `computeDiscards` in `Binder_Lambda` which counts underscores per lambda, `wasUnderscore` in `BindIsOperator`, `WRN_CaseConstantNamedUnderscore`/`WRN_IsTypeNamedUnderscore` in the pattern binders), and in every IDE feature and analyzer that has to ask "is this `_` a symbol or a discard?" (rename, find references, unused-parameter and unused-value analyzers, `IParameterSymbol.IsDiscard`).

**Expected outcome.** `_` always means "I don't care about this value" and never means anything else. All tools could respect that assumption. If `_` is in a location where a type is expected, a compiler error is generated. A developer can read `_` without looking at the surrounding scope. Where someone genuinely needs the name `_` (rare; mostly interop with generated code or a symbol imported from a library) there is an explicit, already-supported spelling, `@_`.

## Detailed design
[design]: #detailed-design

### Principle

1. The token `_` is a *discard-token*. It is **not** an *identifier*.
2. A *discard-token* never binds to, declares, or is looked up as a name. Simple-name lookup never finds a declaration "named `_`" because no declaration can be named by a *discard-token*.
3. Every grammar position that takes an *identifier* continues to accept `@_` (an *identifier* whose name is `_`). Every grammar position that should accept a discard says so explicitly.
4. In a position where a *discard-token* is syntactically present but not permitted, it is a compile-time error. The diagnostic says that `_` is a discard and suggests `@_` if a name was intended.

### Lexical structure and grammar (§6.4.3 Identifiers)

Existing text ([lexical-structure.md](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/lexical-structure.md)):

```antlr
identifier
    : Simple_Identifier
    | contextual_keyword
    | discard_token
    ;

discard_token
    : '_'
    ;
```

Replace with:

```antlr
identifier
    : Simple_Identifier
    | contextual_keyword
    ;

discard_token
    : '_'
    ;
```

and exclude the single character `_` from the unescaped identifier production:

```antlr
fragment Available_Identifier
    // excluding keywords or contextual keywords, and excluding the single character '_' (a discard_token), see note below
    : Basic_Identifier
    ;
```

`Escaped_Identifier` (`'@' Basic_Identifier`) is unchanged, so `@_` remains a *Simple_Identifier* spelled `_`. `__`, `_x`, `x_` etc. are unaffected.

Existing text:

> The semantics of an identifier named `_` depends on the context in which it appears:
>
> - It can denote a named program element, such as a variable, class, or method, or
> - It can denote a discard ([§9.2.9.2](variables.md#9292-discards)).

Replace with:

> The token `_` is a *discard_token* and is not an identifier. It always denotes a discard ([§9.2.9.2](variables.md#9292-discards)) where a *discard_token* is permitted by the grammar and the rules of this specification, and is a compile-time error everywhere else. The identifier `@_` (and any spelling of `_` that uses a *Unicode_Escape_Sequence*, such as `\u005F`) is an ordinary identifier that happens to have the name `_`; it can name any program element and is referred to by `@_`.
>
> *Note*: Because a *discard_token* is not an identifier, `_` and `@_` are **not** "the same identifier" for the purposes of the identifier-equivalence rules above. The `@` prefix is removed from `@_` and the resulting name is `_`, which is also the name that appears in metadata and reflection. *end note*

Today (before this proposal) `@_` and `\u005F` are textually distinct from `_` but resolve to the same name, which is why every existing use of `_` as a name can be mechanically rewritten to `@_` with no change of meaning (see [Compatibility and migration](#compatibility-and-migration)). *end note*

### Where `_` is permitted

| Position | Proposed meaning of `_` |
| --- | --- |
| Discard pattern, subpattern, switch-expression arm (`_` as a *pattern*) | Discard pattern (unchanged), unconditionally |
| *discard_designation* (`T _`, `var _`, `var (_, x)`) in patterns and deconstruction | Discard (unchanged) |
| `out _`, `out T _`, `out var _` | Discard, unconditionally |
| `_ = e;` and `_ = ref e;` | Discard assignment, unconditionally |
| *deconstructor_element* `_` | Discard, unconditionally |
| Declarator in a local variable declaration (`T _ = e;`, `var _ = e;`), `using`, `for` initializer | **Error.** Write `@_`, or use `_ = e;` to discard a result |
| `foreach (T _ in e)` / `foreach (var _ in e)` / `await foreach` | **Error.** Write `@_` |
| `catch (T _)` | **Error.** Write `@_`, or omit the exception variable: `catch (T)` |
| Lambda / anonymous method parameter, any count (`_ => ...`, `(_, x) => ...`, `(int _) => ...`, `delegate (int _) {...}`) | Discard parameter, unconditionally |
| Method, local function, constructor, indexer, operator, delegate, accessor, record/primary-constructor parameter | **Error.** Write `@_` |
| Field, property, event, method, local function, type, type parameter, namespace, enum member, using alias, label, query range variable, tuple element name, named argument name, object/`with` initializer member, member-pattern name, attribute argument name, `x._` member access | **Error.** Write `@_` |
| Anywhere an *expression* is required (`_ + 1`, `Use(_)`, `return _;`, `nameof(_)`, `$"{_}"`, `_ += 1`, `_++`) | **Error** |
| The entire *pattern* of `e is _` or of a switch-statement `case _:` | **Error** (unchanged from the current rule in §11.2.7, now with no name-based exceptions) |

### Simple names and declaration expressions (§12.8.4, §12.19)

§12.8.4 currently ends the simple-name algorithm with:

> - Otherwise, if `e` is zero and `I` is the identifier `_`, the *simple_name* is a *simple discard*, which is a form of declaration expression ([§12.19](expressions.md#1219-declaration-expressions)).
> - Otherwise, the *simple_name* is undefined and a compile-time error occurs.

Remove the first bullet. A *simple_name* is by definition an *identifier* (optionally with type arguments), and `_` is not an identifier, so simple-name lookup can never find, hide, or fall back to a discard.

§12.19 currently says:

> The *simple_name* `_` is also considered a declaration expression if simple name lookup did not find an associated declaration ([§12.8.4](expressions.md#1284-simple-names)). When used as a declaration expression, `_` is called a *simple discard*. It is semantically equivalent to `var _`, but is permitted in more places.

Replace with:

> A *discard_token* in an expression position is a declaration expression called a *simple discard*. It is semantically equivalent to `var _`, but is permitted in more places. A *discard_token* is never reclassified as a *simple_name*, and is never affected by any declaration in scope.

and extend the grammar:

```antlr
declaration_expression
    : local_variable_type (identifier | discard_token)
    | discard_token
    ;
```

The list of syntactic contexts in §12.19 (an `out` *argument_value*, the left side of a simple assignment, a *deconstructor_element*) is unchanged. A simple discard in any other expression position is a compile-time error.

The sentence "A declaration expression with the identifier `_` is a discard" becomes "A declaration expression whose designation is a *discard_token* is a discard". The existing `DeclarationExpressions3` example in §12.19 is shown below; the comments inside the body describe current behavior. Under the proposal, the declaration of `M2` itself is invalid because ordinary method parameters require an identifier:

```csharp
void M1(out int i) { ... }

void M2(string _)   // Proposed: error at the parameter declaration; write @_.
{
    M1(out _);      // Error: `_` is a string
    M1(out var _);
}
```

is replaced by

```csharp
void M1(out int i) { ... }

void M2(string @_)
{
    M1(out _);      // OK: a discard. `@_` is not affected
    M1(out var _);  // OK: a discard
    M1(out @_);     // Error: `@_` is a string
}
```

### Deconstructing assignment (§12.23.3)

Remove the backward-compatibility paragraph:

> For backward compatibility if any *deconstructor_element* is a *discard_token* then:
>
> - If name lookup ([§12.8.4](expressions.md#1284-simple-names)) for “`_`” finds an associated declaration then the *discard_token* is reclassified as a *simple_name*, which is syntactically a *variable_reference*.
> - Otherwise the *discard token* is a simple discard ([§12.19](expressions.md#1219-declaration-expressions)).

Replace with:

> A *discard_token* that is a *deconstructor_element* is always a simple discard ([§12.19](expressions.md#1219-declaration-expressions)). To assign to a variable named `_`, use `@_` (a *variable_reference*).

The statement "It is a compile time error if any *variable_reference*, including any reclassified *discard_token*s, occurring as a *deconstructor_element* is not writeable" drops "including any reclassified *discard_token*s". The grammar disambiguation sentence ("as a *discard_token* and as a *variable_reference*") is no longer needed because the two alternatives can no longer both match a single token.

### Assignment (§12.23)

No grammar change. The statement "The target of a *simple_assignment*, *ref_assignment*, or any target of a *deconstructing_assignment* may be a discard" now means exactly "the target is a *discard_token*". The left operand of a *compound_assignment* shall not be a discard (unchanged), so `_ += 1` is an error whether or not an `@_` exists.

### Discards as variables (§9.2.9.2)

Existing text:

> A ***discard*** is a local variable that has no name. A discard is introduced by a declaration expression ([§12.19](expressions.md#1219-declaration-expressions)) with the identifier `_`; and is either implicitly typed (`_` or `var _`) or explicitly typed (`T _`).
>
> *Note*: `_` is a valid identifier in many forms of declarations. *end note*

Replace with:

> A ***discard*** is a local variable that has no name. A discard is introduced by a declaration expression ([§12.19](expressions.md#1219-declaration-expressions)) ~~with the identifier `_`~~ **with the *discard_token*** **or by a *discard_designation* ([§11.2.3](patterns.md#1123-declaration-pattern))**; and is either implicitly typed (`_` or `var _`) or explicitly typed (`T _`).
>
> *Note*: `_` is never an identifier; a program element that is actually named `_` is declared with `@_`. *end note*

and delete the sentence in the example, "The example assumes that there is no declaration of the name `_` in scope.", which is now vacuous.

### Local variable declarations, `using`, `for` (§13.6.2, §13.14 and §13.9.4)

*Existing grammar* (abbreviated):

```antlr
implicitly_typed_local_variable_declarator
    : identifier '=' expression
    ;

explicitly_typed_local_variable_declarator
    : identifier ('=' local_variable_initializer)?
    ;
```

No grammar change. These declarators continue to require an *identifier*. Because `_` is not an identifier, `int _ = e;`, `var _ = e;`, `using var _ = e;`, `using (var _ = e) { }`, `await using var _ = e;`, and `for (int _ = 0; ... )` are compile-time errors at the declaration, even if the variable would never be read.

`ref_local_variable_declarator`, `constant_declarator`, `fixed`, `static` locals and `scoped` locals also keep `identifier`: `ref var _ = ref x;` and `const int _ = 1;` are errors. Use `@_` to declare a named variable or resource. To evaluate an expression and discard its result without a variable declaration, use `_ = e;` (or `_ = ref x;` for a ref discard assignment).

### `foreach` (§13.9.5)

Existing grammar:

```antlr
foreach_statement
    : // synchronous foreach
      'foreach' '(' ref_kind? local_variable_type identifier 'in' expression ')'
          embedded_statement
    | // asynchronous foreach
      'await' 'foreach' '(' local_variable_type identifier 'in' expression ')'
          embedded_statement
    | // deconstructing foreach
      'await'? 'foreach' '(' deconstructor 'in' expression ')'
          embedded_statement
    ;
```

No grammar change. The non-deconstructing forms continue to require an *identifier*, so `foreach (T _ in e)`, `foreach (var _ in e)`, and their asynchronous or `ref` forms are compile-time errors. Write `@_` for a named iteration variable. Discards within a deconstructing `foreach`, such as `foreach (var (_, item) in e)`, remain permitted. A bare `_` in `foreach (_ in e)` is also invalid.

### Exception specifier (§13.11)

The grammar `exception_specifier : '(' type identifier? ')'` is unchanged. `catch (E _)` is a compile-time error because `_` is not an identifier. Use `catch (E @_)` for a named exception variable, or `catch (E)` when no exception variable is needed.

### Anonymous functions (§12.21.2) and parameter scopes (§7.7)

Existing grammar uses `identifier` for `explicit_anonymous_function_parameter` and `implicit_anonymous_function_parameter`. New:

```antlr
explicit_anonymous_function_parameter
    : anonymous_function_parameter_modifier? type (identifier | discard_token)
    ;

implicit_anonymous_function_parameter
    : identifier
    | discard_token
    ;
```

The existing csharplang spec for [lambda discard parameters](csharp-9.0/lambda-discard-parameters.md) says:

> In the parameter list of a lambda or anonymous method with more than one parameter named `_`, such parameters are discard parameters.
> Note: if a single parameter is named `_` then it is a regular parameter for backwards compatibility reasons.

Replace with:

> A parameter of a lambda or anonymous method whose name is the *discard_token* is a ***discard parameter***, regardless of how many such parameters the parameter list contains. A discard parameter has the declared (or inferred) type and modifiers as any other parameter, but introduces no name to any scope.

and update the two scope sentences (§7.7 and the lambda-discard-parameters proposal)

> With the exception of discard parameters, the scope of a parameter declared in a *lambda_expression* is the *anonymous_function_body* of that *lambda_expression*.

so that they no longer need a count. The "Simple names" change of the lambda-discard-parameters proposal ("... parameter (with the exception of discard parameters) ...") is subsumed: a *simple_name* cannot be `_`.

The rule in §12.21.2 that "it is a compile-time error for the name of a parameter of the anonymous function to match the name of a local variable, local constant or parameter whose scope includes the ..." is unchanged and never applies to discard parameters.

**Method parameters.** Methods, local functions, constructors, indexers, operators, delegates, accessors and record/primary-constructor parameters keep `identifier` in the grammar. `_` as a parameter name there is an error ("'_' is a discard; use '@_' to name a parameter '_'"). See [Open questions](#open-questions) for allowing discard parameters on local functions.

### Patterns (§11.2)

Existing grammar for designations:

```antlr
simple_designation
    : discard_designation
    | single_variable_designation
    ;
discard_designation
    : '_'
    ;
single_variable_designation
    : identifier
    ;
```

is unchanged (`identifier` no longer includes `_`, so the "when both alternatives are applicable the former shall be chosen" sentence and its note become unnecessary and are deleted).

§11.2.7 *Discard pattern* currently reads:

> Where the syntactic context permits a *pattern*, if the token `_` would resolve as a *simple_name* ([§12.8.4](expressions.md#1284-simple-names)) to an accessible constant or to a type, then `_` is *not* treated as a *discard_pattern*. Instead:
>
> - If `_` resolves to an accessible constant, the `_` is interpreted as a *constant_pattern* ([§11.2.2](patterns.md#1122-constant-pattern)) whose constant expression is that constant.
> - If `_` resolves to a type, then in the right-hand side of an `is` operator the construct *relational_expression* `is _` is interpreted as the is-type operator ([§12.14.12.1](expressions.md#1214121-the-is-type-operator)) testing against that type. In any other syntactic context that admits a *pattern*, a bare `_` resolving to a type is not by itself a valid *pattern*; however, `_` may appear as the *type* of a *declaration_pattern* (e.g., `_ x`) or in other pattern forms that explicitly name a type.
>
> This rule preserves backward compatibility with code that defined `_` as a type or identifier prior to the introduction of the discard pattern. If `_` resolves to anything other than an accessible constant or type (for example, a local variable, parameter, field, or method), the rule does not apply and `_` remains a *discard_pattern*.
>
> *Note*: This is analogous to the rule for `var` in [§11.2.4](patterns.md#1124-var-pattern), except that for `_` an in-scope constant or type causes `_` to be interpreted as a reference to that declaration rather than producing an error. *end note*
>
> If, after applying the preceding rule, the token `_` is still a *discard_pattern*, it is a compile-time error for that *discard_pattern* to appear as the entire *pattern* of a *relational_expression* of the form *relational_expression* `is` *pattern*, or as the entire *pattern* of a *switch_label*. A *discard_pattern* may, however, appear as a *subpattern* of an enclosing pattern (for example, as a *subpattern* of a *positional_pattern* or *property_pattern*).

Replace the whole passage (keeping the final "*Note*: ... use a *var_pattern* with a discard `var _`") with:

> The token `_` where a *pattern* is expected is always a *discard_pattern*. It is never interpreted as a constant, a type, or any other name: a constant named `_` is written `@_` (a *constant_pattern*), and a type named `_` is written `@_` (a *type_pattern* / the type of a *declaration_pattern*).
>
> It is a compile-time error for a *discard_pattern* to appear as the entire *pattern* of a *relational_expression* of the form *relational_expression* `is` *pattern*, or as the entire *pattern* of a *switch_label*. A *discard_pattern* may appear as a *subpattern* of an enclosing pattern, as an arm of a *switch_expression*, and under `not`/`and`/`or` combinators.
>
> *Note*: To match any value in those two positions, use `var _`. *end note*

The `WithoutUnderscore`/`WithUnderscore` example in §11.2.7 (in which an in-scope constant `_` changes the meaning of the `_` arm) is replaced by an example showing that `const int @_ = 0;` has no effect on `_ => ...` and that `@_ => ...` matches the constant.

§11.2.3 note:

> *Note*: This treatment of `_` within a *declaration_pattern* differs from that of a standalone `_` written as a *pattern* ([§11.2.7](patterns.md#1127-discard-pattern)): in the latter case, an in-scope constant or type named `_`, if any, is *not* hidden. *end note*

is deleted: there is no longer any difference.

### Names and types (§7.8, §12.8.7 and others)

No text is needed beyond the grammar change, because they all take an *identifier*: *namespace_or_type_name*, *type_parameter*, *member_access*, *named_argument*, *tuple_element*, *label*, *object_initializer* members, *member_declarator*, *property_subpattern* names, *query* range variables, `nameof`, interpolation holes, attribute arguments. `_` in any of those is a syntax or semantic error with the diagnostic described in the Principle, and `@_` works:

```csharp
class @_ { public int @_; }            // type and field named '_'
@_ v = new @_ { @_ = 1 };
var s = nameof(@_);                    // "_"
if (o is @_ x) { }                     // declaration pattern on the type named '_' (today only this spelling parses)
var t = (@_: 1, b: 2); var one = t.@_; // tuple element named '_'
```

### Diagnostics

- **New error**: *"'_' is a discard and cannot be used here. Use '@_' to refer to or declare a program element named '_'."* reported (a) for a *discard_token* where an *expression* or *identifier* is required, (b) for the parameter positions in the table above, and (c) for the other declaration positions.
- `WRN_CaseConstantNamedUnderscore` (CS8512) and `WRN_IsTypeNamedUnderscore` (CS8513) become unreachable and are retired (the compiler's existing guidance for both already is "use `var _` to discard, or `@_` to refer to the declaration").
- A declaration requiring an identifier is an error when written with `_`, with or without an initializer (`int _;`, `int _ = e;`, `var _ = e;`). Lambda and anonymous-method discard parameters, declaration expressions in `out` arguments, and discard designations in patterns and deconstruction are permitted as described above.
- The unused-local, unused-parameter, and `IDE0059`/`IDE0060` analyzers need no special handling for `_` any more.

### Implementation notes (Roslyn)

The contextual `SyntaxKind.UnderscoreToken` already identifies the token (`IsUnderscoreToken`), so the lexer/parser work is mostly to make it *non*-contextual and recognize it unconditionally in the permitted discard positions, including single-parameter lambdas and anonymous methods. In the binder, `FallBackOnDiscard` (Binder_Expressions.cs) and the `wasUnderscore` special-casing in `BindIsOperator` (Binder_Operators.cs), the `underscoresCount` logic in Binder_Lambda.cs, and `reportIfConstantNamedUnderscore` (SwitchBinder_Patterns.cs) disappear or collapse. Ordinary variable, iteration-variable, resource, exception-variable, and member-parameter declarations continue to require identifiers; there are no new discard-local lowering rules. This is an estimate for discussion; it has not been prototyped.

## Breaking changes
[breaking-changes]: #breaking-changes

The baseline for these breaking changes is .NET 11, RC2. The *proposed* column is a description, except where a "migrated" form is shown, which compiles and runs today with identical meaning. In every case where a source program changes meaning, the proposal makes it a **compile-time error** rather than a silent change, with the narrow exception called out in section D. `@_` is already accepted by every compiler version, so the migration `_ → @_` is safe to apply before upgrading.

### A. A local named `_` is declared: becomes an error at the declaration

```csharp
// Before: local named '_'                         // After: error CS-new at the declaration of '_'
int _ = Compute();                                  // error: a local declaration requires an identifier
Console.WriteLine(_);                               // error: '_' is a discard
// Migrated:
int @_ = Compute();
Console.WriteLine(@_);
```

The same declaration error applies to `var _ = e;`, `for (int _ = 0; _ < n; _++)`, `foreach (var _ in xs)`, `using var _ = s;`, `catch (Exception _)`, `from _ in xs select _ * 2`, and `let _ = ...`, whether or not the declared variable is subsequently read.

### B. A name `_` is declared in a position where discards are not allowed: error at the declaration

```csharp
// Before                                  // After
void M(int _) => Use(_);                   // error at the parameter; migrated: void M(int @_) => Use(@_);
int _;  /  void _() {}  /  class _ {}  /  enum E { _ }  /  record R(int _);  /  class C(int _);
const int _ = 1;
using _ = System.Console;
var t = (_: 1, b: 2); t._
goto _; _: ;
M(_: 1);   new C { _ = 1 };   x is { _: 1 };   x is (_: 1, b: 2)
void M<_>() {}   List<_>   _ v = new _();
```

all become errors and need `@_`. `o is _ x`, `case _ x:` and the like never parsed with a type `_`; they continue to need `@_ x`.

### C. Lambda parameters

```csharp
Func<int, int> f   = _ => _ + 1;            // Before: OK (parameter named '_').    After: error at the '_' in the body.
Func<int, int, int> g = (_, x) => _ + x;    // Before: OK.                          After: error at the '_' in the body.
Func<int, int> h   = _ => 0;                // Before: parameter named '_'.         After: discard parameter. No source change needed.
Func<int, int, int> k = (_, _) => 0;        // Before: discards.                    After: discards (unchanged).
Func<int, int, int> m = (_, @_) => @_;      // Before: CS0100/CS0229.               After: OK: first is a discard, second is a parameter named '_'.
// Migrated forms of the first two:
Func<int, int> f2 = @_ => @_ + 1;
Func<int, int, int> g2 = (@_, x) => @_ + x;
```

A lambda such as `_ => { _ = 5; }` (assignment to the parameter) changes from "assignment to the parameter" to "discard assignment". Any subsequent read of that parameter using `_` is a compile-time error, as in the first two examples above.

### D. Meaning of `out _`, `_ = e`, `(_, x) = e` when a program element named `_` exists

Today these bind to an in-scope variable named `_` if there is one; under the proposal they are always discards.

```csharp
int _ = -1;                  // today: local; proposal: error at the declaration (write @_ to retain the variable)
int.TryParse("5", out _);    // today: writes the local (`_` is now 5)        proposal: discard; local (if it existed) is untouched
_ = 5;                       // today: assignment to the local                 proposal: discard assignment
(_, x) = (1, 2);             // today: assigns the local                       proposal: discards the first element
```

- If the element named `_` is declared in the *same source* with the spelling `_`, its declaration is a compile-time error (section A/B), so the unchanged program fails to compile rather than silently discarding assignments to that variable.
- **Silent change (narrow):** if the element is *not* declared with the bare spelling `_` in the same compilation, the meaning of `out _`/`_ = e`/`(_, x) = e`/`case _:`/`is _` silently changes. This happens (1) when the symbol comes from metadata or a base class in another assembly (a field, constant, type or method named `_`), and (2) when the existing declaration was already spelled `@_` or `\u005F` and later references use the bare `_` (these are textually different spellings that bind to the same thing today). The first case relies on third-party libraries exposing public/protected members named `_`; the second on programmers having already opted into the `@_` spelling. The migration analyzer (below) can detect both: it flags any bare `_` whose simple-name lookup finds a symbol today.

### E. Patterns involving a constant or a type named `_`

| Code | Before | After |
| --- | --- | --- |
| `switch (x) { case _: }` with a constant `_` | Constant pattern (warning CS8512) | Error: bare `_` as the whole `case` pattern. Fix: `case @_:` (constant) or `case var _:` (discard) |
| `switch (x) { case _: }` with a type `_` | Type pattern | Error. Fix: `case @_:` |
| `x is _` with a type `_` | Is-type test (warning CS8513) | Error. Fix: `x is @_` (type) or `x is var _` |
| `x is _` with a constant `_` | Error (CS0246/CS0118; the standard draft incorrectly claims a constant pattern) | Error (`x is @_` works) |
| `x switch { _ => ... }` with a constant, local or field `_` | Discard arm | Discard arm (unchanged) |
| `(a, b) is (_, _)`, `s is { P: _ }` with a constant `_` | Discard subpatterns | Discard subpatterns (unchanged) |
| `o is _ x` with a type `_` | Parse error (the standard incorrectly says allowed) | Parse error; `@_ x` required (unchanged) |

### F. Things that stop being errors (not breaking, listed for completeness)

```csharp
Func<int, int, int> m = (_, @_) => @_;       // Before: CS0100/CS0229. After: first parameter is a discard; second is named '_'.
```

Repeated declarations such as `var _ = F(); var _ = G();`, nested `foreach (var _ in ...)` statements, and repeated `using var _ = ...` declarations remain errors; each declaration requires an identifier.

### G. Reflection, metadata and public API

- A lambda parameter written `_` is already emitted with the metadata name `_` (both parameters of `(_, _) => 0` and the single parameter of `_ => 0` report `ParameterInfo.Name == "_"`), so runtime reflection output does not change.
- A member declared `@_` has the metadata name `_`, exactly as `_` has today.
- `IParameterSymbol.IsDiscard` becomes `true` for every lambda/anonymous-method `_` parameter, not only when there are two or more.

### H. A type named `_` (any kind)

Under the rule "`_` is never an identifier", a type named `_` stops existing as far as the spelling `_` is concerned, so the blast radius is every *use* of the type, not only the declaration.

| Code (type `_` in scope) | Before | After (as written) | Fix |
| --- | --- | --- | --- |
| `class _ {}` / `struct _` / `interface _` / `enum _` / `delegate void _()` / `record _` / `record struct _` / `class _<T>` / nested `class _` / `namespace _` / `static class _` | Declares the type | Error at the declaration | `@_` |
| `using _ = System.Text.StringBuilder;` | Alias declared | Error at the declaration | `using @_ = ...;` |
| `class C<_> { _ v; }` / `void M<_>()` | Type parameter, used as a type | Error at the declaration | `@_` |
| `_ x;`, `_ x = default;`, `_ _x`, `_[]`, `_?`, `ref _ r`, `foreach (_ x in e)`, `catch (_ e)`, `using (_ x = e)`, `(_ a, _ b) => ...`, member/parameter/return types, `class D : _`, `where T : _` | Bind to the type | Error: `_` is not a type | `@_` |
| `(_)o`, `new _()`, `typeof(_)`, `default(_)`, `nameof(_)`, `List<_>`, `stackalloc _[2]` | Bind to the type | Error | `@_` |
| `_.Foo`, `_.Foo.Bar` (static member, nested type, namespace member, "Color Color") | Binds to the type/namespace | Error | `@_.Foo` |
| `using static _;`, `using _;` (namespace), `[_]` where the attribute is `class _ : Attribute` | Binds | Error | `@_` |
| `x is _` (warning CS8513) / `case _:` (type pattern) | Type test | Error (already in E) | `@_` |
| `x is _ y`, `case _ y:`, `_ { }`, `_(1)`, `x is _<int>`, `(_ a, _ b)` patterns, `{ P: _ y }` | Syntax error (verified for every kind) | Syntax error | `@_` already works in all of them |
| `x is _ or int`, `x is (_)`, `x is not _`, `{ } and _`, `_ =>` arm, `(_, _)` / `{ P: _ }` subpatterns | Discard (the type is ignored) | Discard (unchanged) | none |
| `out _`, `_ = e`, `_ += 1`, `(_, x) = e`, `(_, _) = pair`, `(_ a, _) = pair` | Error CS0118 (CS0305 if `_<T>`): simple-name lookup finds the type and the discard is only a fallback when nothing is found | Valid discard | none (not a break) |
| `[_]` where the class is named `_Attribute` | Binds to `_Attribute` through the attribute-suffix probe | Error: `_` is no longer a name | `[_Attribute]`. `[@_]` does **not** work: `@` turns off the suffix probe (CS0246, verified) |

Findings specific to types:

- **No new silent meaning change for types.** All breaks are loud errors at the declaration or at the use. The only silent case is the one already in D (the type is declared `@_` or comes from metadata, and bare `_` is later written in `out _`/`_ = e`/`(_, x) = e`).
- A lambda parameter `_` already hides the type inside the lambda body today, and a local `int _` hides the type for `_.Foo` (CS1061) but not in type-only positions, so the "both a type and a name `_`" programs are already fragile.
- The migration is mechanical. Each kind can be migrated by rewriting every occurrence to `@_` and the output matches. A blind textual rewrite is not safe in patterns: `@_ or int` is the type pattern, whereas `_ or int` is a discard, so the migration tool must use binding information (the same constraint as section D).
- Prevalence: see [Type `_`](#type-_).

### Compatibility and migration
[migration]: #compatibility-and-migration

Because `@_` is already an ordinary identifier spelled `_` and resolves to the same thing as a bare `_` in every current language version, the migration is mechanical and can be applied *before* a project opts into the new rules:

1. For each existing declaration whose name is `_`, rewrite the declaration and every reference to `@_`, unless it is a lambda or anonymous-method parameter that is never referenced by name. Rewrite ordinary declarations even if the declared variable is never read. This includes `using`, `foreach`, and `catch` variables and ordinary method/member parameters.
2. For an unused local whose only purpose is to evaluate an expression, `var _ = F();` can instead become `_ = F();`. Keep a named resource variable (`using var @_ = F();`) when disposal is required.
3. Leave `out _`, `_ = e`, `(_, x) = e`, `_ =>` in switch expressions, `out var _` and every other discard alone.

A Roslyn analyzer and code fix is part of the design (see [Open questions](#open-questions)): *"The name `_` is a discard under C# N; write `@_`"*. It should report on every simple-name lookup of `_` that currently resolves to a symbol or whose declaration is named `_` in a position where a discard is not allowed, and offer a fix-all (per document/project/solution) that rewrites the declaration and its references to `@_`. Source generators and generated code are the most likely sources of name `_` and should be covered by analyzer severity configuration rather than hand edits.

## Type `_`
[type-underscore]: #type-_

**Proposed:** `_` is neither a type nor a variable. A type or namespace named `_` is declared and referenced as `@_`, exactly like any other program element (table H above). The alternative that keeps `_` legal as a type name is recorded under [Alternatives](#alternatives).

### Evidence

1. **Parsing today.** In declaration statements `_ x;`, `_ _x`, `_ _`, `(_)o`, `new _()`, `typeof(_)`, `List<_>`, `foreach (_ x in e)`, `catch (_ e)`, `using (_ x = e)`, typed lambda parameters and `(_ a, _ b) = pair` all parse and bind to a type `_`. The earlier statement that `_ x`/`_ _x` do not parse is true only in *pattern* positions (`o is _ y`, `case _ y:`, `_ { }`, `(_ a, _ b)` patterns, `{ P: _ y }`, `o is _.Foo`, `o is _<int>`), where every kind of type gives a syntax error. Roslyn: `LooksLikeTypeOfPattern` excludes the underscore token, and `ParsePrimaryPattern` returns a discard pattern for `_` unconditionally.
2. **Binding today.** Discards are a *fallback* used only when simple-name lookup of `_` finds nothing, so with a type (or namespace) `_` in scope `out _`, `_ = e`, `(_, x) = e` fail with CS0118 (CS0305 for `_<T>`). Under this proposal a discard never looks anything up, which removes the lookup-then-fallback.
3. **Patterns.** With a type `_` in scope the bare `_` in `or`/`and`/`not`/parenthesised patterns, subpatterns and switch-expression arms is already a discard, while `o is _` (CS8513) and the switch-statement `case _:` are the type. Under this proposal both become errors and `wasUnderscore`/CS8512/CS8513 are retired.
4. **Migration is mechanical.** Every kind migrated to `@_` gives identical output (`existing-tests.cs` vs `new-behavior-tests.cs` section 11). Extra break: `[_]` bound to `class _Attribute` must become `[_Attribute]` (`[@_]` does not probe the suffix).

Prevalence (limits stated): I could not measure how common a type named `_` is. GitHub code search through `gh api search/code` reports unreliable total counts (for `"class _"` 346112 hits, of which only 21 of the first 100 contained a real `class _` declaration in the returned fragments), and it cannot be restricted to declarations. What I did verify from fragments: type `_` exists in real code, including two widely used libraries: nested sink classes `private/internal sealed class _ : Sink<...>` used as `new _(this)` in dotnet/reactive (Rx.NET; the search for `"class _ :" repo:dotnet/reactive` reported 107 matching files, not individually verified) and in AvaloniaUI/Avalonia (3 files returned by `"class _ :" repo:AvaloniaUI/Avalonia`); `enum _ {` in generated code (`2881099/dotnetGen_*`) and others; `struct _ {` (about 6 hits); `interface _ {` (1 hit); and a handful of `class _ {` in application repos. Conclusion: rare overall, not absent, concentrated in generated code and in at least two popular libraries.

## Drawbacks
[drawbacks]: #drawbacks

- It is a breaking change. Usage of `_` as a real name is uncommon but not rare (single-parameter lambdas such as `_ => _.Foo`, `foreach (var _ in ...)` bodies that read `_`, test frameworks, DSLs, generated code and the "I named my parameter `_` because I don't care about it but then I used it" case).
- The behavior of an unchanged-looking program differs depending on the language version (see the gating question), so `_` read-compatible code needs a one-time rewrite when a project upgrades.
- Existing idioms such as `var _ = F();`, `foreach (var _ in e)`, and `using var _ = F();` become declaration errors even when the variable is never read. They require migration to `@_` or, for an unused result, to discard assignment.
- The Roslyn public API and third-party analyzers that special-case `_` (rename, unused-value, naming rules) need updates.

## Alternatives
[alternatives]: #alternatives

- **Keep `_` legal as a type (or namespace) name.** `_` may still name a type or namespace, declared and referenced in type/namespace positions, but can never be a variable, constant, member or other identifier in an expression. Discards follow the rules in this proposal: `_` in `out _`, `_ = e`, `(_, x) = e`, `_ =>` and pattern discards is always a discard.
  - *Evidence:* declarations and expressions that use a type `_` already parse today (`_ x;`, `new _()`, `typeof(_)`, `List<_>`, `(_)o`), so X needs no new parsing there; in patterns `_ x`, `_.Foo`, `_ { }` stay errors unless a one-token lookahead is added (optional). Type `_` exists in real code (see [Type `_`](#type-_)).
  - *Benefits:* no source break for type `_`, including public types that consumers reference and generated code; no `[_]` to `[_Attribute]` rewrite.
  - *Costs:* `_` is a name in type space but not in value space (a two-level rule); the binder needs context-dependent lookup (discard position vs. type position); `is _` and `case _:` remain ambiguous, so either the type interpretation and warnings CS8512/CS8513 are kept, or they become errors anyway; the grammar needs a type-name `_` distinct from *identifier*, plus a restricted simple name in expressions for `_.Foo`, `nameof(_)`, `(_)e`, `new _()`.
  - *Why not chosen:* it is the only option that leaves `_` with two meanings, which the proposal exists to remove, and the zero-break benefit applies to a rare pattern. This decision can be relaxed later by allowing type `_`; the reverse (tightening) is not possible, so this keeps that door open.

Comparison of allowing `_` for a type and this proposal:

| Criterion | Type `_` allowed | This proposal  |
| --- | --- | --- |
| Rule to teach | "`_` is a discard everywhere except where a type name is expected" | "`_` is never a name" |
| Grammar | Needs a type-name `_` distinct from *identifier* (type contexts, `namespace_or_type_name`) plus a restricted simple-name in expressions for `_.Foo`, `nameof(_)`, `(_)e`, `new _()`; "Color Color" `_ _ = new _()` stays legal | One token rule; no special case |
| Parser | Declarations/expressions work today. Patterns: `_ x`, `_.Foo` stay errors unless a lookahead is added | No change from this proposal |
| Binder | Context-dependent lookup (discard position vs type position); removes lookup-then-fallback | Discard never looks anything up; removes `wasUnderscore` and the CS8512/CS8513 special cases |
| `is _` / `case _:` | Still ambiguous (type test or error): a decision with trade-offs either way | Error, consistent with the constant case |
| Source breaks for type `_` | None | Every declaration and every use (table H) |
| Migration | None for types | `@_` everywhere; analyzer/fix-all; `[_]` to `[_Attribute]` |
| Consistency | `_` is a name in type space but not in value space (two-level rule) | Uniform |

- **Status quo plus warnings.** C# already warns when `_` hides a constant in `case _:` or a type in `is _` (CS8512, CS8513). Adding more warnings (for a named local `_` that is read, a single `_` lambda parameter that is read, `out _` that binds to a variable) addresses the symptoms but leaves two rules and the standard/compiler disagreements in place.
- **Make `_` a reserved keyword.** In effect this is what the proposal does for the single token `_` (it cannot be an identifier without `@`), but without introducing a keyword that would also affect `__`, `_x` or contextual uses.
- **Restrict only *new* declaration kinds (no source breaks).** E.g. declare `_` as a discard in any construct added after C# N. This keeps today's rules and adds a third one.
- **Make single-`_` lambda parameters discards but keep everything else (only fixes lambdas).** The lambda counting rule is the least defensible inconsistency, but it is only one of many.
- **A different escape than `@_`.** Rejected: `@_` already exists, already means this, and needs no new token.
- **Allow discard declarations and additional discard-variable forms.** Permit `T _ = e;`, `var _ = e;`, `foreach (T _ in e)`, `using var _ = e;`, and `catch (T _)` without introducing a name. This would preserve some existing unused-variable idioms, but adds declaration forms and resource-lifetime rules beyond the existing discard contexts. The proposal instead makes these declarations errors.

## Open questions
[open]: #open-questions

1. **Language-version gating (not specified).** Options to discuss: (a) the new rules apply only under a new language version and older versions keep today's behavior (the migration to `@_` compiles in every version, so projects can fix code before upgrading); (b) a *warning wave* first: in older versions warn (opt-in via analyzer/warning level) wherever the new rules would change or break the program, then break in a later version; (c) no gating (not recommended). Which version, and whether any piece (for example lambda discard parameters regardless of count) can ship earlier and independently, are undecided.
2. **Analyzer and code fix.** Which analyzer ID/package (compiler warning, IDE analyzer, `dotnet format`)? What severity by default in the old language version? Should the fix also be offered to rename to a descriptive name rather than `@_` for locals/parameters that are read? How are generated files handled?
3. **Method-like parameters.** Should local functions (and possibly methods whose parameters are never exposed by name, such as private methods and lambdas' method-group counterparts) accept discard parameters (`void F(int _, int _)`)? The proposal says no, because named arguments, overrides and metadata see parameter names.
4. **Bare `_` as the whole pattern of `is` / `case`.** The proposal keeps it an error (as today) to avoid a silent change from the current constant/type interpretation. Once the old interpretations are gone it could become legal (`x is _` always true), at the cost of making a no-op expression easy to write.
5. **Other binding positions that today use an identifier**: query range variables (`from _ in xs from _ in ys`), tuple element names, named arguments. Should any of them accept discards, or is `@_` the answer? (The proposal says `@_`.)
6. **`\u005F`.** The compiler today treats `\u005F` as a plain identifier (not the discard token), so it is already an escape for `_`. The proposal keeps that; should it be documented, restricted or deprecated?
7. **Silent change in section D.** Is a warning in older language versions on "bare `_` binds to a symbol" sufficient, or should such a binding be an error under the new rules even before the declaration is migrated?
8. **Public API.** Whether `SyntaxFacts.IsValidIdentifier("_")`, `SyntaxKind.UnderscoreToken` (currently contextual) and `ISymbol.Name == "_"` semantics change, and how analyzers written against today's API are told.
9. **Standard alignment.** The current standard text for §11.2.7 does not match Roslyn (see Motivation). The proposal's replacement text is intentionally Roslyn-compatible for the cases that remain; whether the ECMA/ISO text should be corrected independently now.

## Test programs
[test-programs]: #test-programs

Two single-file programs accompany this proposal (outside this repository; to be shared in the discussion). Both run with `dotnet run <file>.cs` on the .NET SDK 11 preview:

* `existing-tests.cs` — demonstrates every current behavior listed in the Motivation, computing the observation at run time (section 11 covers a type named `_` of every kind in every syntactic position), plus a documented list of snippets that do not compile (with the diagnostic seen) because they cannot coexist in one file.
* `new-behavior-tests.cs` — for each of those cases shows what the proposal does (a runnable "migrated" form wherever the behavior can be expressed today, since `@_` is available today; a clearly marked "describe-only" note where the new behavior cannot be executed because the proposal is not implemented).
