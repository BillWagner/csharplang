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

```csharp
// Phase 1: how `_` is treated TODAY (C# 14 / Roslyn in .NET SDK 11 preview).
// Run with:  dotnet run existing-tests.cs
//
// Every line of output is "<case>: <observation>".  For each case the observation is
// computed at run time (e.g. "binds to the local" vs "discard"), not asserted by hand.
//
// Cases that cannot compile together with the rest, or that do not compile at all, are
// listed in the comment block at the bottom ("NOT RUNNABLE") together with the compiler
// diagnostic that was observed when each snippet was compiled on its own.
//
// Warnings CS8512/CS8513 (constant/type named `_` in a pattern) are intentionally left
// enabled-but-suppressed below so that the output stays readable.

#:property AllowUnsafeBlocks=true
#pragma warning disable CS8512, CS8513, CS0219, CS0168, CS8321, CS0414, CS0649, CS0169, CS0162, CS8794, CS8632

using System;
using System.Collections.Generic;
using System.Linq;

Console.WriteLine("=== 1. Declarations that make `_` a real name ===");
Declarations.Run();

Console.WriteLine("\n=== 2. Lambda and anonymous-method parameters ===");
Lambdas.Run();

Console.WriteLine("\n=== 3. out arguments ===");
OutArguments.Run();

Console.WriteLine("\n=== 4. Assignment and deconstruction ===");
Assignments.Run();

Console.WriteLine("\n=== 5. Patterns (is / switch / subpatterns / designations) ===");
Patterns.Run();

Console.WriteLine("\n=== 6. nameof / interpolation / member access ===");
NamesAndStrings.Run();

Console.WriteLine("\n=== 7. A type named `_` (each case in its own namespace so `_` is only visible there) ===");
TypeNamedUnderscore.Run();

Console.WriteLine("\n=== 8. A constant named `_` ===");
ConstNamedUnderscore.Run();

Console.WriteLine("\n=== 9. Members named `_` ===");
MemberNamedUnderscore.Run();

Console.WriteLine("\n=== 10. `@_` today ===");
AtUnderscore.Run();

Console.WriteLine("\n=== 11. A TYPE named `_` of every kind, in every syntactic position that compiles (namespaces K1..K15) ===");
TypeKinds.Run();

static class Out
{
    public static void Show(string label, object? value) => Console.WriteLine($"{label,-58}: {value}");
}

// ---------------------------------------------------------------------------------------
// 1. Declarations. In all of these `_` is an ordinary identifier: it declares a named
//    program element, it participates in scope rules, and later uses of `_` bind to it.
// ---------------------------------------------------------------------------------------
static class Declarations
{
    public static void Run()
    {
        { int _ = 1; Out.Show("1a  `int _ = 1;` then read `_`", $"named local, value {_}"); }
        { var _ = 2; Out.Show("1b  `var _ = 2;` (declaration STATEMENT) then read `_`", $"named local, value {_}"); }
        { int _ = 1; _ = 3; Out.Show("1c  `int _ = 1; _ = 3;`", $"assignment to the local, now {_}"); }
        {
            int sum = 0;
            foreach (var _ in new[] { 10, 20 }) sum += _;
            Out.Show("1d  `foreach (var _ in ...)` body reads `_`", $"named iteration variable, sum {sum}");
        }
        {
            int sum = 0;
            for (int _ = 0; _ < 3; _++) sum += _;
            Out.Show("1e  `for (int _ = 0; ...)`", $"named loop variable, sum {sum}");
        }
        {
            using var _ = new System.IO.MemoryStream(new byte[4]);
            Out.Show("1f  `using var _ = ...;` then `_.Length`", $"named local, Length {_.Length}");
        }
        {
            try { throw new InvalidOperationException("boom"); }
            catch (Exception _) { Out.Show("1g  `catch (Exception _)` then `_.Message`", $"named exception variable, {_.Message}"); }
        }
        Out.Show("1h  method parameter `int _`", MethodParam(41));
        {
            int _() => 5;
            Out.Show("1i  local function named `_`", $"called `_()` = {_()}");
        }
        {
            var q = from _ in new[] { 1, 2, 3 } select _ * 2;
            Out.Show("1j  query range variable `from _ in ...`", string.Join(",", q));
        }
        {
            var q = from x in new[] { 1, 2, 3 } let _ = x * 10 select _;
            Out.Show("1k  query `let _ = ...`", string.Join(",", q));
        }
        {
            var t = (_: 1, b: 2);
            Out.Show("1l  tuple element named `_`", $"t._ = {t._}");
        }
        {
            Out.Show("1m  named argument `_:`", NamedArg(_: 7));
        }
        {
            goto _;
        _:
            Out.Show("1n  label named `_`", "goto _ / _: works");
        }
    }

    static string MethodParam(int _) => $"named parameter, value {_}";
    static string NamedArg(int _) => $"named parameter, value {_}";
}

// ---------------------------------------------------------------------------------------
// 2. Lambdas. A SINGLE `_` parameter is an ordinary parameter (compat). TWO OR MORE `_`
//    parameters in the same list are discards. Whether a `_` is a name therefore depends on
//    how many other `_` are in the same parameter list (Binder_Lambda.cs: underscoresCount).
// ---------------------------------------------------------------------------------------
static class Lambdas
{
    public static void Run()
    {
        Func<int, int> one = _ => _ + 1;
        Out.Show("2a  `_ => _ + 1`", $"`_` is a real parameter, f(1) = {one(1)}");

        Func<int, int> oneParen = (_) => _ + 1;
        Out.Show("2b  `(_) => _ + 1`", $"`_` is a real parameter, f(1) = {oneParen(1)}");

        Func<int, int> oneTyped = (int _) => _ + 1;
        Out.Show("2c  `(int _) => _ + 1`", $"`_` is a real parameter, f(1) = {oneTyped(1)}");

        Func<int, int> anon = delegate (int _) { return _ + 1; };
        Out.Show("2d  `delegate (int _) { return _ + 1; }`", $"`_` is a real parameter, f(1) = {anon(1)}");

        Func<int, int, int> mixed = (_, x) => _ + x;
        Out.Show("2e  `(_, x) => _ + x`", $"ONE `_` => real parameter, f(1,2) = {mixed(1, 2)}");

        Func<int, int, int> two = (_, _) => 0;
        Out.Show("2f  `(_, _) => 0`", "two `_` => both are discards (no names introduced)");

        Func<int, int, int, int> three = (_, x, _) => x;
        Out.Show("2g  `(_, x, _) => x`", "two `_` => discards, `x` is a real parameter");

        Func<int, int, int> typedTwo = (int _, int _) => 0;
        Out.Show("2h  `(int _, int _) => 0`", "two `_` => discards");

        Func<int, int, int> anonTwo = delegate (int _, int _) { return 0; };
        Out.Show("2i  `delegate (int _, int _) { return 0; }`", "two `_` => discards");

        // With discards, a `_` in the body is NOT the parameter: it binds to whatever is in
        // scope outside (here a string local), proving the parameters introduced no name.
        string _ = "outer-local";
        Func<int, int, string> discardsSeeOuter = (_, _) => _;
        Out.Show("2j  `(_, _) => _` with an outer local `_`", $"body binds to the OUTER `_`: \"{discardsSeeOuter(1, 2)}\"");

        Func<int, string> singleHidesOuter = _ => _.ToString();
        Out.Show("2k  `_ => _.ToString()` with an outer local `_`", $"parameter hides the outer `_`: \"{singleHidesOuter(5)}\"");

        Func<int, int, int> nestedLocal = (_, _) => { int _ = 3; return _; };
        Out.Show("2l  `(_, _) => { int _ = 3; return _; }`", $"discard params don't conflict with a body local `_`: {nestedLocal(0, 0)}");

        string outer2 = "unchanged";
        Func<int, int, bool> outInBody = (_, _) => int.TryParse("12", out var _);
        Out.Show("2m  `(_, _) => int.TryParse(\"12\", out var _)`", $"discard inside a discard lambda: {outInBody(0, 0)}");

        Action<int> assignInBody = _ => { _ = 5; Out.Show("2n  `_ => { _ = 5; ... }`", $"assigns to the single parameter (now {_})"); };
        assignInBody(1);

        Func<int, Func<int, int, int>> nested = _ => (_, _) => 0;
        Out.Show("2o  `_ => (_, _) => 0`", "inner two `_` are discards; they do not conflict with outer param `_`");

        var withDefaults = (int _ = 1, int _ = 2) => 0;
        Out.Show("2p  `(int _ = 1, int _ = 2) => 0`", "two `_` => discards (also valid with default values)");
    }
}

// ---------------------------------------------------------------------------------------
// 3. out arguments. `out _` is a discard only if simple-name lookup of `_` fails.
//    `out var _` and `out T _` are ALWAYS discards.
// ---------------------------------------------------------------------------------------
static class OutArguments
{
    public static void Run()
    {
        {
            int.TryParse("5", out _);
            Out.Show("3a  `out _` with no `_` in scope", "discard (the call compiles; `_` is not readable afterwards)");
        }
        {
            int _ = -1;
            int.TryParse("5", out _);
            Out.Show("3b  `int _ = -1; ... out _`", $"binds to the LOCAL: `_` is now {_}");
        }
        {
            int _ = -1;
            int.TryParse("5", out var _);
            Out.Show("3c  `int _ = -1; ... out var _`", $"discard: local `_` unchanged = {_}");
        }
        {
            int _ = -1;
            int.TryParse("5", out int _);
            Out.Show("3d  `int _ = -1; ... out int _`", $"discard: local `_` unchanged = {_}");
        }
        {
            string _ = "text";
            int.TryParse("5", out var _);
            Out.Show("3e  `string _ ...; out var _`", "discard, even though `_` is a string local (the `out _` form would be CS1503)");
        }
    }
}

// ---------------------------------------------------------------------------------------
// 4. Assignments / deconstruction.
// ---------------------------------------------------------------------------------------
static class Assignments
{
    public static void Run()
    {
        {
            _ = 5;
            Out.Show("4a  `_ = 5;` with no `_` in scope", "discard assignment (compiles; `_` isn't readable)");
        }
        {
            int _ = 0;
            _ = 5;
            Out.Show("4b  `int _ = 0; _ = 5;`", $"assigns to the local: `_` = {_}");
        }
        {
            int x;
            (_, x) = (1, 2);
            Out.Show("4c  `(_, x) = (1, 2);` with no `_` in scope", $"first element discarded, x = {x}");
        }
        {
            int _ = 0; int x;
            (_, x) = (1, 2);
            Out.Show("4d  `int _ = 0; (_, x) = (1, 2);`", $"first element assigned to the LOCAL: `_` = {_}, x = {x}");
        }
        {
            int _ = 0;
            var (_, y) = (1, 2);
            Out.Show("4e  `int _ = 0; var (_, y) = (1, 2);`", $"`var (_, y)` is a discard: local `_` still {_}, y = {y}");
        }
        {
            int _ = 0;
            (int _, int z) = (1, 2);
            Out.Show("4f  `int _ = 0; (int _, int z) = (1, 2);`", $"typed designation is a discard: local `_` still {_}, z = {z}");
        }
        {
            int _ = 0; int x;
            (int a, var _, _) = (1, 2, 3);
            Out.Show("4g  `(int a, var _, _) = (1, 2, 3);` with local `_`", $"`var _` discard, bare `_` assigns the LOCAL: `_` = {_}, a = {a}");
        }
        {
            int x = 0;
            ref int r = ref x;
            _ = ref x;
            Out.Show("4h  `_ = ref x;` with no `_` in scope", "ref discard assignment compiles");
        }
        {
            int x = 0;
            ref var _ = ref x;
            _ = 9;
            Out.Show("4i  `ref var _ = ref x;` (declaration STATEMENT)", $"named ref local: writes through, x = {x}");
        }
        {
            int _ = 0;
            _ += 4;
            Out.Show("4j  `int _ = 0; _ += 4;`", $"compound assignment on the named local = {_}");
        }
        {
            var results = new List<(int, int)> { (1, 2) };
            foreach (var (_, b) in results) Out.Show("4k  `foreach (var (_, b) in ...)`", $"discard + b = {b}");
        }
    }
}

// ---------------------------------------------------------------------------------------
// 5. Patterns.
// ---------------------------------------------------------------------------------------
static class Patterns
{
    public static void Run()
    {
        object o = 42;
        {
            var r = o is int _;
            Out.Show("5a  `o is int _`", $"declaration pattern with discard designation: {r}");
        }
        {
            int _ = 0;
            var r = o is int _;
            Out.Show("5b  `int _ = 0; o is int _`", $"discard designation, no conflict with local: {r}");
        }
        {
            var r = o is var _;
            Out.Show("5c  `o is var _`", $"var pattern with discard designation: {r}");
        }
        {
            if (o is int n && n > 0) Out.Show("5d  (control) `o is int n`", "named designation");
        }
        {
            var t = (1, 2);
            Out.Show("5e  `t is (_, _)`", t is (_, _));
        }
        {
            var s = "abc";
            Out.Show("5f  `s is { Length: _ }`", s is { Length: _ });
        }
        {
            int[] a = { 1, 2 };
            Out.Show("5g  `a is [_, ..]`", a is [_, ..]);
        }
        {
            int i = 5;
            var r = i switch { 1 => "one", _ => "other" };
            Out.Show("5h  switch expression `_ =>` (no `_` in scope)", $"discard pattern -> {r}");
        }
        {
            int _ = 1; // a LOCAL (not a constant)
            int i = 5;
            var r = i switch { 1 => "one", _ => "other" };
            Out.Show("5i  switch expression `_ =>` with a local `_` in scope", $"still the discard pattern -> {r}");
        }
        {
            int i = 5;
            switch (i)
            {
                case var _: Out.Show("5j  switch statement `case var _:`", "matches everything"); break;
            }
        }
        {
            int i = 5;
            switch (i)
            {
                case int _: Out.Show("5k  switch statement `case int _:`", "type pattern + discard designation"); break;
            }
        }
        {
            var t = (1, 2);
            int _ = 7;
            var r = t is (_, _);
            Out.Show("5l  `t is (_, _)` with a local `_`", $"still discard subpatterns -> {r}");
        }
        {
            var t = (_: 1, b: 2);
            Out.Show("5m  `t is (_: 1, b: 2)`", $"tuple element NAME `_` in a subpattern label -> {t is (_: 1, b: 2)}");
        }
    }
}

// ---------------------------------------------------------------------------------------
// 6. nameof, interpolation, member access.
// ---------------------------------------------------------------------------------------
static class NamesAndStrings
{
    public static void Run()
    {
        {
            int _ = 3;
            Out.Show("6a  `nameof(_)` with a local `_`", nameof(_));
            Out.Show("6b  `$\"{_}\"` with a local `_`", $"{_}");
        }
        {
            var t = (_: "item", b: 2);
            Out.Show("6c  `t._`", t._);
        }
    }
}

// ---------------------------------------------------------------------------------------
// 7. A TYPE named `_`.  Each case lives in its own namespace.
// ---------------------------------------------------------------------------------------
static class TypeNamedUnderscore
{
    public static void Run()
    {
        T1.X.Run(); T2.X.Run(); T3.X.Run(); T4.X.Run(); T5.X.Run(); T6.X.Run();
    }
}

namespace T1
{
    class _ { }
    static class X
    {
        public static void Run()
        {
            object o = new _();
            object p = "not an _";
            Out.Show("7a  type `_`: `o is _`", $"is-TYPE test (warning CS8513 compile-time): new _() -> {o is _}, string -> {p is _}");
        }
    }
}
namespace T2
{
    class _ { }
    static class X
    {
        public static void Run()
        {
            object p = "not an _";
            var r = p switch { _ => "arm `_` matched" };
            Out.Show("7b  type `_`: switch expression arm `_ =>`", $"{r}  (a string matched => it is the DISCARD pattern, not type `_`)");
        }
    }
}
namespace T3
{
    class _ { }
    static class X
    {
        public static void Run()
        {
            object p = "not an _";
            string r = "no match";
            switch (p) { case _: r = "case `_` matched"; break; }
            Out.Show("7c  type `_`: switch statement `case _:`", $"{r}  (a string did not match => it is the TYPE pattern `_`; in a switch STATEMENT type `_` wins)");
        }
    }
}
namespace T4
{
    class _ { }
    class D { public object P = "str"; }
    static class X
    {
        public static void Run()
        {
            var d = new D();
            Out.Show("7d  type `_`: property subpattern `{ P: _ }`", $"{(d is { P: _ })}  (P is a string => DISCARD pattern)");
        }
    }
}
namespace T5
{
    class _ { public override string ToString() => "an _"; }
    static class X
    {
        public static void Run()
        {
            object o = new _();
            // `@_` is the only way to spell the type in a declaration pattern: `_ x` does not parse (see NOT RUNNABLE).
            if (o is @_ x) Out.Show("7e  type `_`: `o is @_ x`", $"declaration pattern on type `_` via `@_` -> {x}");
            if (o is @_) Out.Show("7f  type `_`: `o is @_`", "is-type test via `@_`");
            var r = o switch { @_ => "type-`_` pattern", _ => "discard" };
            Out.Show("7g  type `_`: switch arm `@_ =>`", r);
        }
    }
}
namespace T6
{
    class _ { public string Hi() => "hi from type _"; }
    static class X
    {
        public static void Run()
        {
            _ v = new _();
            Out.Show("7h  type `_` as a normal type name", $"`_ v = new _();` -> {v.Hi()}");
            var list = new List<_> { new _() };
            Out.Show("7i  `List<_>`", $"type argument `_` -> Count {list.Count}");
            int.TryParse("1", out var _);
            Out.Show("7j  type `_` + `out var _`", "discard (type in scope doesn't matter for `var _`)");
        }
    }
}

// ---------------------------------------------------------------------------------------
// 8. A CONSTANT named `_`.
// ---------------------------------------------------------------------------------------
static class ConstNamedUnderscore
{
    const int _ = 1;

    public static void Run()
    {
        Out.Show("8a  const `_ = 1`: `1 switch { 2 => \"two\", _ => \"arm _\" }`", 1 switch { 2 => "two", _ => "arm _ (either the constant or the discard: ambiguous, see 8b)" });
        Out.Show("8b  const `_ = 1`: `5 switch { 2 => \"two\", _ => \"arm _\" }` (5 != 1)", Try(() => 5 switch { 2 => "two", _ => "arm _" }) + "  <- 5 matched `_`, so in a switch EXPRESSION the arm is the DISCARD pattern even though a constant `_` is in scope");
        Out.Show("8c  const `_ = 1`: `5 switch { @_ => \"const\", var _ => \"discard\" }`", 5 switch { @_ => "const", var _ => "discard" });
        string r = "none";
        switch (5) { case _: r = "case _ matched"; break; default: r = "case _ did NOT match 5 => in a switch STATEMENT the constant wins: constant pattern (warning CS8512)"; break; }
        Out.Show("8d  const `_ = 1`: `switch (5) { case _: .. }`", r);
        Out.Show("8e  const `_ = 1`: subpatterns `(5, 6) is (_, _)`", $"{(5, 6) is (_, _)}  <- True: discard subpatterns (constant ignored)");
        Out.Show("8f  const `_ = 1`: subpatterns `(1, 6) is (_, _)`", (1, 6) is (_, _));
        Out.Show("8g  const `_ = 1`: `\"s\" is { Length: _ }` (Length 1)", "s" is { Length: _ });
        Out.Show("8h  const `_ = 1`: `\"ss\" is { Length: _ }` (Length 2)", $"{"ss" is { Length: _ }}  <- True: discard subpattern");
    }

    static string Try(Func<string> f)
    {
        try { return f(); } catch (Exception e) { return e.GetType().Name; }
    }
}

// ---------------------------------------------------------------------------------------
// 9. MEMBERS named `_` (fields, methods, properties, enum members, record parameters ...).
// ---------------------------------------------------------------------------------------
static class MemberNamedUnderscore
{
    public static void Run()
    {
        var c = new HasFieldUnderscore();
        Out.Show("9a  field `_`: `_ = 5;` inside the class", c.AssignViaSimpleName());
        Out.Show("9b  field `_`: out `_` inside the class", c.OutViaSimpleName());
        Out.Show("9c  field `_`: deconstruct `(_, x) = (1, 2)` inside the class", c.DeconstructViaSimpleName());
        Out.Show("9d  field `_`: `_` as a switch-expression arm inside the class", c.SwitchArm());
        Out.Show("9e  method `_()` : call `_()`", new HasMethodUnderscore().Call());
        Out.Show("9f  enum member `E._`", MyEnum._);
        Out.Show("9g  record parameter `_`", new Rec(3)._);
        Out.Show("9h  primary-constructor parameter `_`", new Prim(4).Get());
        Out.Show("9i  object initializer `new C { _ = 6 }`", new HasPublicUnderscore { _ = 6 }._);
        Out.Show("9j  member pattern `{ _: 6 }`", new HasPublicUnderscore { _ = 6 } is { _: 6 });
    }

    class HasFieldUnderscore
    {
        int _;
        public string AssignViaSimpleName() { _ = 5; return $"`_ = 5` assigned the FIELD: {_}"; }
        public string OutViaSimpleName() { int.TryParse("8", out _); return $"`out _` wrote the FIELD: {_}"; }
        public string DeconstructViaSimpleName() { int x; (_, x) = (11, 2); return $"deconstruction wrote the FIELD: {_}"; }
        public string SwitchArm() => 99 switch { 1 => "one", _ => "discard arm (field `_` is NOT a constant, so the arm stays a discard)" };
    }
    class HasMethodUnderscore { void _() { } public string Call() { _(); return "called method `_()`"; } }
    enum MyEnum { _ }
    record Rec(int _);
    class Prim(int _) { public int Get() => _; }
    class HasPublicUnderscore { public int _; }
}

// ---------------------------------------------------------------------------------------
// 10. `@_` TODAY: it is just an identifier whose spelling is `_` ("@" is removed), so it is
//     the same name as `_` in a declaration and in simple-name lookup.
// ---------------------------------------------------------------------------------------
static class AtUnderscore
{
    public static void Run()
    {
        {
            int @_ = 5;
            Out.Show("10a `int @_ = 5;` then `_`", $"same name: `_` reads {_}");
        }
        {
            Func<int, int> f = @_ => _ + 1;
            Out.Show("10b `@_ => _ + 1`", $"real parameter, f(1) = {f(1)}");
        }
        {
            int @_ = 1;
            int.TryParse("9", out @_);
            Out.Show("10c `out @_` with a local", $"writes the local: {_}");
        }
        {
            // For the two-underscore lambda rule, `@_` does NOT count as an underscore (it is not a
            // contextual-keyword token), so `(_, @_) => ...` has ONE `_` (real param) and ONE `@_`
            // (real param): duplicate parameter => CS0100 (see NOT RUNNABLE).
            Out.Show("10d `(_, @_) =>`", "CS0100 duplicate parameter today (see NOT RUNNABLE)");
        }
    }
}

// ---------------------------------------------------------------------------------------
// 11. A TYPE named `_`.  Every kind of type is declared in its own namespace (K1..K15) so `_` is only a type
//     inside that namespace.  Only positions that COMPILE are executed here; every position that does not
//     compile for a type `_` is listed (with the observed diagnostic) in the "TYPE `_` MATRIX" comment at the bottom.
//     Key facts these cases demonstrate:
//       * `_ x`, `_ _x`, `_ _`, `(_)o`, `new _()`, `typeof(_)`, `default(_)`, `nameof(_)`, `List<_>`, `_.Foo`,
//         `o is _`, `catch (_ e)`, `foreach (_ e in ..)`, `using (_ u = ..)`, `(_ a, _ b) =>`, `(_ a, _ b) = e` all bind `_` as the TYPE.
//       * `o is _ or int`, `o is (_)`, `x switch { _ => .. }`, `(_, _)`/`{ P: _ }` subpatterns are the DISCARD pattern
//         even though a type `_` is in scope (11A-4, 11A-5, 11A-7, 11A-8). A switch STATEMENT `case _:` is the type pattern (11A-6).
//       * Inside the body of `_ => ...` the lambda parameter hides the type (11A-9, 11E-2).
// ---------------------------------------------------------------------------------------
static class TypeKinds
{
    public static void Run()
    {
        K1.X.Run(); K2.X.Run(); K3.X.Run(); K4.X.Run(); K5.X.Run(); K6.X.Run(); K7.X.Run();
        K8.X.Run(); K9.X.Run(); K10.X.Run(); K11.X.Run(); K12.X.Run(); K13.X.Run(); K14.X.Run(); K15.X.Run();
    }
}

namespace K1 // 11A: class
{
    class _ : Exception, IDisposable
    {
        public static int Foo = 7;
        public int P = 3;
        public void Dispose() { }
    }

    static class X
    {
        static void Q(out _ p) => p = new _();

        public static void Run()
        {
            _ x = new _();
            object o = x, s = "a string";
            Out.Show("11A-1 class: decl / cast / new / typeof / nameof", $"`_ x`={x.P}; `(_)o`={((_)o).P}; `typeof(_)`={typeof(_).Name}; `nameof(_)`={nameof(_)}; `_.Foo`={_.Foo}");
            Out.Show("11A-2 class: generic args / arrays / default", $"`List<_>`={new List<_> { x }.Count}; `new _[2]`={new _[2].Length}; `default(_)` is null={default(_) is null}");
            Out.Show("11A-3 class: `is _` (string, `_` instance)", $"{s is _}, {o is _}  (type test)");
            Out.Show("11A-4 class: `is _ or int` / `is (_)` (string)", $"{s is _ or int}, {s is (_)}  (true => `_` is the DISCARD here, not the type)");
            Out.Show("11A-5 class: `s is { } and _` / `is int or _`", $"{s is { } and _}, {s is int or _}  (true => discard)");
            string r = "no match";
            switch (s) { case _: r = "matched"; break; }
            Out.Show("11A-6 class: statement `case _:` (string)", $"{r}  (not matched => TYPE pattern)");
            Out.Show("11A-7 class: switch expression arm `_ =>` (string)", s switch { _ => "matched" } + "  (matched => DISCARD)");
            Out.Show("11A-8 class: positional/property subpattern `_`", $"{(s, 1) is (_, _)}, {(P: s, Q: 1) is { P: _ }}  (discard subpatterns)");
            Func<_, _, int> f = (_ a, _ b) => a.P + b.P;
            Func<_, int> g = _ => _.P;
            Out.Show("11A-9 class: lambda `(_ a, _ b)` / `_ => _.P`", $"{f(x, x)}, {g(x)}  (the lambda parameter `_` hides the type inside the body)");
            (_, _) pair = (x, x); (_ a, _ b) = pair;
            Out.Show("11A-10 class: deconstruction `(_ a, _ b) = pair`", $"{a.P + b.P}");
            _ q; Q(out q);
            Q(out _ q2);
            Out.Show("11A-11 class: `Q(out _ q)` typed out declaration", $"{q.P}, {q2.P}");
            int n = 0; foreach (_ e in new[] { x, x }) n += e.P;
            Out.Show("11A-12 class: `foreach (_ e in ...)`", n);
            try { throw new _(); } catch (_ e) { Out.Show("11A-13 class: `catch (_ e)`", e.GetType().Name); }
            try { throw new _(); } catch (_) { Out.Show("11A-14 class: `catch (_)`", "caught"); }
            using (_ u = new _()) { Out.Show("11A-15 class: `using (_ u = new _())`", "ok"); }
            {
                _ _ = new _();
                Out.Show("11A-16 class: Color Color `_ _ = new _();` `_.Foo`, `_.P`", $"{_.Foo}, {_.P}");
            }
            _ _x = new _(); _ x2 = new(); var xs = new _[] { new _() };
            Out.Show("11A-17 class: `_ _x`, target-typed `new()`, `new _[] {..}`", $"{_x.P}, {x2.P}, {xs.Length}");
            Out.Show("11A-18 class: `o as _`, `typeof(_[])`", $"{(o as _)?.P}, {typeof(_[]).Name}");
        }
    }
}

namespace K2 // 11B: struct
{
    struct _ { public static int Foo = 7; public int P; }

    static class X
    {
        public static void Run()
        {
            _ x = new _ { P = 3 };
            object o = x, s = "a string";
            Span<_> sp = stackalloc _[2];
            _? n = null;
            Out.Show("11B-1 struct: `_ x`, `_.Foo`, `Span<_> = stackalloc _[2]`, `_?`", $"{x.P}, {_.Foo}, {sp.Length}, {n.HasValue}");
            Out.Show("11B-2 struct: `o is _`, `s is _`, `(_)o`, `default(_)`", $"{o is _}, {s is _}, {((_)o).P}, {default(_).P}");
            unsafe { _* p = &x; Out.Show("11B-3 struct: `_* p`, `sizeof(_)`", $"{p->P}, {sizeof(_)}"); }
        }
    }
}

namespace K3 // 11C: interface
{
    interface _ { static int Foo = 7; int P { get; } }
    class Impl : _ { public int P => 3; }
    class D : _ { public int P => 4; }

    static class X
    {
        public static void Run()
        {
            _ i = new Impl();
            object o = i, s = "a string";
            Out.Show("11C-1 interface: `_ i`, `_.Foo`, `o is _`, `s is _`, `(_)o`", $"{i.P}, {_.Foo}, {o is _}, {s is _}, {((_)o).P}");
            Out.Show("11C-2 interface: `class D : _`", new D().P);
        }
    }
}

namespace K4 // 11D: enum
{
    enum _ { Foo = 1, Bar = 2 }

    static class X
    {
        public static void Run()
        {
            _ e = _.Foo; const _ k = _.Bar; object o = e, s = "a string";
            var c = (_)2;
            Out.Show("11D-1 enum: `_ e = _.Foo`, `const _ k`, `(_)2`, `e == _.Foo`", $"{e}, {k}, {c}, {e == _.Foo}");
            Out.Show("11D-2 enum: `o is _`, `s is _`, `typeof(_)`, `nameof(_.Foo)`", $"{o is _}, {s is _}, {typeof(_).Name}, {nameof(_.Foo)}");
            Out.Show("11D-3 enum: `Enum.GetValues<_>()`", string.Join(",", Enum.GetValues<_>()));
        }
    }
}

namespace K5 // 11E: delegate
{
    delegate int _(int a);

    static class X
    {
        public static void Run()
        {
            _ d = a => a + 1; _ d2 = new _(a => a + 2);
            object o = d, s = "a string";
            Out.Show("11E-1 delegate: `_ d = a => ...`, `new _(...)`, `o is _`, `s is _`", $"{d(1)}, {d2(1)}, {o is _}, {s is _}");
            Func<int, int> f = _ => _ + 1;
            Out.Show("11E-2 delegate: `Func<int,int> f = _ => _ + 1` (lambda param hides type)", f(1));
        }
    }
}

namespace K6 // 11F: record
{
    record _(int P) { public static int Foo = 7; }

    static class X
    {
        public static void Run()
        {
            _ x = new _(3); object o = x, s = "a string";
            var y = x with { P = 4 };
            Out.Show("11F-1 record: `new _(3)`, `with`, `_.Foo`, `o is _`, `s is _`, `(_)o`", $"{x.P}, {y.P}, {_.Foo}, {o is _}, {s is _}, {((_)o).P}");
            Out.Show("11F-2 record: `typeof(_)`, `nameof(_)`, ToString", $"{typeof(_).Name}, {nameof(_)}, {x}");
        }
    }
}

namespace K7 // 11G: record struct
{
    record struct _(int P) { public static int Foo = 7; }

    static class X
    {
        public static void Run()
        {
            _ x = new _(3); object o = x, s = "a string";
            Out.Show("11G-1 record struct: `new _(3)`, `_.Foo`, `o is _`, `s is _`, `(_)o`", $"{x.P}, {_.Foo}, {o is _}, {s is _}, {((_)o).P}");
        }
    }
}

namespace K8 // 11H: generic type `_<T>`
{
    class _<T> { public static int Foo = 7; public T? V; }

    static class X
    {
        public static void Run()
        {
            _<int> x = new _<int> { V = 3 }; object o = x, s = "a string";
            Out.Show("11H-1 generic: `_<int> x`, `new _<int>()`, `_<int>.Foo`, `typeof(_<>)`", $"{x.V}, {_<int>.Foo}, {typeof(_<>).Name}");
            Out.Show("11H-2 generic: `o is _<int>`", "does not parse (CS1525); see TYPE `_` MATRIX. Using `@_<int>`: " + (o is @_<int>) + ", " + (s is @_<int>));
        }
    }
}

namespace K9 // 11I: using alias `_`
{
    using _ = System.Text.StringBuilder;

    static class X
    {
        public static void Run()
        {
            _ sb = new _("ab"); object o = sb, s = "a string";
            Out.Show("11I-1 alias: `_ sb = new _(..)`, `o is _`, `s is _`, `(_)o`, `typeof(_)`", $"{sb.Length}, {o is _}, {s is _}, {((_)o).Length}, {typeof(_).Name}");
            Out.Show("11I-2 alias: `nameof(_)`", nameof(_));
        }
    }
}

namespace K10 // 11J: nested type
{
    static class X
    {
        class _ { public static int Foo = 7; public int P = 3; }

        public static void Run()
        {
            _ x = new _(); object o = x, s = "a string";
            Out.Show("11J-1 nested: `_ x = new _()`, `_.Foo`, `o is _`, `s is _`, `typeof(_).FullName`", $"{x.P}, {_.Foo}, {o is _}, {s is _}, {typeof(_).FullName}");
        }
    }
}

namespace K11 // 11K: type parameter `_`
{
    static class X
    {
        static string G<_>(object o) where _ : new()
        {
            _ v = new _(); var d = default(_);
            return $"`_ v = new _()`={v}; `typeof(_)`={typeof(_).Name}; `o is _`={o is _}; `nameof(_)`={nameof(_)}";
        }

        static string H<_>(object o) => $"{((_)o)}, {o is _}, {o is _ or int}";

        public static void Run()
        {
            Out.Show("11K-1 type param: `G<_>` body uses `_` as a type", G<object>(1));
            Out.Show("11K-2 type param: `H<_>` `(_)o`, `o is _`, `o is _ or int`", H<string>("s"));
        }
    }
}

namespace K12 // 11L: namespace `_`
{
    namespace _ { public class Foo { public static int Bar = 7; } }
    namespace Use
    {
        using _;
        static class X
        {
            public static void Run()
            {
                var a = new _.Foo(); var b = new Foo();
                Out.Show("11L-1 namespace: `new _.Foo()`, `using _;` + `Foo`, `_.Foo.Bar`, `nameof(_.Foo)`", $"{a.GetType().FullName}, {b.GetType().Name}, {_.Foo.Bar}, {nameof(_.Foo)}");
                Out.Show("11L-2 namespace: `nameof(_)`", nameof(_));
            }
        }
    }
    static class X { public static void Run() => Use.X.Run(); }
}

namespace K13 // 11M: `using static _;` and extension methods in a static class `_`
{
    static class _ { public static int Foo = 7; public static int Ext(this int x) => x + 1; public class Inner { } }
    namespace Use
    {
        using static _;
        static class X
        {
            public static void Run()
            {
                Out.Show("11M-1 `using static _;` then `Foo`, `new Inner()`", $"{Foo}, {new Inner().GetType().Name}");
                Out.Show("11M-2 extension method in static class `_`: `1.Ext()`", 1.Ext());
            }
        }
    }
    static class X { public static void Run() => Use.X.Run(); }
}

namespace K14 // 11N: attribute class `_`
{
    class _ : Attribute { }
    [_] class A1 { }
    [@_] class A2 { }
    [_()] class A3 { }

    static class X
    {
        public static void Run()
        {
            Out.Show("11N-1 attribute class `_`: `[_]` binds to", string.Join(",", typeof(A1).GetCustomAttributes(false).Select(a => a.GetType().Name)));
            Out.Show("11N-2 attribute class `_`: `[@_]` binds to", string.Join(",", typeof(A2).GetCustomAttributes(false).Select(a => a.GetType().Name)));
        }
    }
}

namespace K15 // 11O: attribute class named `_Attribute`, referenced as `[_]` (a name that is NOT `_`)
{
    class _Attribute : Attribute { }
    [_] class A1 { }
    [_Attribute] class A2 { }

    static class X
    {
        public static void Run()
        {
            Out.Show("11O-1 attribute class `_Attribute`: `[_]` binds to", string.Join(",", typeof(A1).GetCustomAttributes(false).Select(a => a.GetType().Name)));
            Out.Show("11O-2 attribute class `_Attribute`: `[_Attribute]` binds to", string.Join(",", typeof(A2).GetCustomAttributes(false).Select(a => a.GetType().Name)));
        }
    }
}

// =======================================================================================
// TYPE `_` MATRIX: every position below was compiled on its own (csc, -langversion:preview) against a type `_` of each
// kind: C=class S=struct I=interface E=enum D=delegate R=record RS=record struct G=generic `_<T>` A=using alias
// N=namespace `_` SC=static class T=attribute class.  Result "ok" means it compiles and `_` is the TYPE (or NS).
// Diagnostics: CS0118 "'_' is a type but is used like a variable", CS0305 generic arity, CS1002/CS1003 syntax error.
// Per-kind semantic errors that do not depend on `_` (e.g. `new _()` on an interface, `catch (_)` on a struct,
// `stackalloc` of a class, static class as a variable type) are not listed.
// =======================================================================================
//
// -- Positions that bind `_` as the TYPE (ok for C,S,I,E,D,R,RS,A,T unless a kind-specific reason is given) -----------------
//   _ x = default;   _ x;   _ _x = default;   _ _ = default;   var f = _.Foo;        decl / Color Color
//   var x = (_)o;    var x = new _();    new _ { };   _ x = new();   typeof(_)   typeof(_[])   default(_)
//   Func<_, _, int> f = (_ a, _ b) => 0;   Func<_, int> f = _ => 0;   delegate (_ a) { return 1; }
//   List<_> l;   new List<_>();   Gen<_>();   _[] a = new _[2];   _? n;   (_, int) t;   (_ a, int b) t;   ref _ r = ref l;
//   foreach (_ x in ..)   foreach (_ _ in ..)   foreach (_ _x in ..)   using (_ x = ..)   using _ x = ..;   using var _ = new _();
//   catch (_ e)   catch (_)   catch (_ _)          (class deriving from Exception only)
//   Span<_> sp = stackalloc _[2]   (struct, enum, record struct)       unsafe { _* p; sizeof(_) }
//   (_ a, _ b) = pair;   Q(out _ x);   method/field/property/parameter/return types `_ f;` `static _ M()` `M(_ p)` `M(_ _)` `M(_ _x)`
//   _ L(_ a) => a;  (local function)   class D : _ {}   explicit operator _   where T : _   new _[] { }   o as _  (class-like kinds)
//   [_] class A {}   (when `_` is an attribute class)       [return: _]       typeof(_) in attribute arguments
//   o is _   (type test; warning CS8513 for class/struct/...)        switch statement `case _:` (type pattern)       (_) as cast target
//   nameof(_)   (namespace `_` too)       static members `_.Foo`       `using static _;`       `using A = _;`
//   Generic `_<T>`: `_<int> v`, `new _<int>()`, `_<int>.Foo`, `typeof(_<>)` are ok; the non-generic uses all give CS0305 (see below).
//   Type parameter `_` (`M<_>()`, `class C<_>`): `_ v = new _()`, `default(_)`, `typeof(_)`, `o is _`, `case _:`, `(_)o`, `_.Foo` (static abstract) are ok.
//
// -- Syntax errors for a type `_` (independent of the kind; the pattern parser never starts a type with `_`) --------------
//   o is _ y        error CS1002       o is _ _        error CS1002       o is _ _x       error CS1002
//   o is _ { }      error CS1002       o is _ { P: 1 } error CS1002       o is _(1)       error CS1002 (positional/recursive on a type `_`)
//   o is (_ y)      error CS1003       o is _<int>     error CS1525       o is _<int> y   error CS1525
//   o is _.Foo      error CS1002/CS1003  (enum constant via a type `_`)     case _.Foo:     error CS1003
//   e switch { _.Foo => 1, _ => 2 }    error CS1003/CS1525/CS8504
//   case _ x:  case _ _x:   error CS1003          case _<int> y:   error CS1002,CS1003,CS1513,CS1525          (`case _ when true:` is ok: `_` is the type)
//   o switch { _ x => 1 }  o switch { _ _x => 1 }  o switch { _ { } => 1 }    error CS1003
//   tp is { Item1: _ x }   tp is (_ a, _ b)      error CS1003
//   `@_` works in every one of these (`o is @_ y`, `o is @_ { P: 1 }`, `o is @_(1)`, `o is @_<int> y`, `case @_ x:`).
//
// -- A type `_` makes today's discards FAIL (`_` binds to the type before the discard fallback) -------------------------
//   int.TryParse("1", out _);                   error CS0118 for C,S,I,E,D,R,RS,A,N,SC,T; CS0305 for generic `_<T>`
//   _ = o;      _ += 1;      (_, x) = tp;       error CS0118 / CS0305 (same set)       (_, _) = pair; (_ a, _) = pair;   error CS0118
//   nested type `_`:  int.TryParse("1", out _); _ = 1;      error CS0118       using _ = System.Int32; out _ / _ = 1;  error CS0118
//   type parameter `_`: out _ , _ = 1                        error CS0118       namespace `_`: out _ , _ = 1             error CS0118
//   Reason (Binder_Expressions.cs): the discard is only a FALLBACK (`FallBackOnDiscard`) taken when simple-name lookup of `_` fails.
//   `var (_, x) = tp`, `out var _`, `out int _`, `foreach (var _ in ..)`, `var _ = 1;`, `x switch { _ => .. }`, `tp is (_, _)`
//   and `tp is { Item1: _ }` are not affected (they are discard DESIGNATIONS/PATTERNS, never looked up).
//
// -- `_` in expression position when a type `_` exists ----------------------------------------------------------------
//   var q = _;  $"{_}"  _ * 2  _ < i                 error CS0119 ('_' is a type, which is not valid in the given context)
//   _[0] = 1;                                         error CS0119
//   (_) + 1 (cast-or-paren)                           error CS0119 ('(_)' is a parenthesized type)      (_)-1  error CS0075
//   var v = _.Foo.Bar;  (namespace `_`)               ok       (namespace `_` + `o is _`  error CS0118)
//   int _ = 0; _.Foo                                  error CS1061 (a local `_` of another type wins; the type `_` is hidden)
//   `_ _ = new _(); _.Foo` and `_.P`                  ok (Color Color)
//
// -- Generic type `_<T>` --------------------------------------------------------------------------------------------------
//   every use of `_` without type arguments (`_ x`, `new _()`, `typeof(_)`, `(_)o`, `out _`, `_ = e` ...)   error CS0305
//   `var _ = 1;`, `x switch { _ => .. }`, `tp is (_, _)`, `o is (_)`, `o is _ or int`, `nameof(_)`(CS0305), lambda `_ => ..` (ok)
//   `_<int> v;` on non-generic kinds                  error CS0308 (non-generic type used with type arguments); alias/namespace `_<int>`: CS0307
//
// -- Namespace `_` --------------------------------------------------------------------------------------------------------
//   `_ x;` `(_)o` `typeof(_)` `default(_)` `new _()` `out _` `_ = 1` `o is _` `case _:` `List<_>` ...   error CS0118 ('_' is a namespace but is used like a type/variable)
//   `_.Foo x;` `new _.Foo()` `new global::_.Foo()` `using _;` `namespace _.X {}` `nameof(_)` `_.Foo.Bar` (expression)   ok
//
// -- Static class `_` -----------------------------------------------------------------------------------------------------
//   every use as a variable/field/parameter/array/type argument type   error CS0723 / CS0721 / CS0719 / CS0718 (static types); `_.Foo`, `typeof(_)`, `o is _`, `using static _;`, extension `1.Ext()`  ok
//
// -- Both a type `_` and a name `_` in scope ------------------------------------------------------------------------------
//   int _ = 0; _ x = default;   var x = (_)o;   new _();   typeof(_);   o is _;      all ok  (type contexts never see the local `_`)
//   var _ = 1; var x = new _();                       ok
//   foreach (var _ in arr) { new _(); }               ok
//   int _ = 0; var x = _.Foo;                         error CS1061 (an expression `_.Foo` finds the local first)
//
// -- Attributes -----------------------------------------------------------------------------------------------------------
//   class _ : Attribute: [_] ok, [@_] ok, [_()] ok, [return: _] ok   (see 11N)
//   class _Attribute : Attribute: [_] binds to _Attribute (11O-1); [@_] error CS0246 (the verbatim `@` turns off the `Attribute` suffix probing);
//                                 [_Attribute] ok (11O-2)
//   Non-attribute type `_`: [_] error CS0616 (class/struct/enum/...) / CS0653 (interface)
//
// -- Declaring a type `_` -------------------------------------------------------------------------------------------------
//   class/struct/interface/enum/delegate/record/record struct/class _<T>/using _ = X;/namespace _/type parameter `_`/nested type `_`   all ok
//   class _ { int _; }  /  class _ { _ _; }   error CS0542 (member name equals enclosing type name; unrelated to the ambiguity)
//
// =======================================================================================
// NOT RUNNABLE: snippets verified by compiling each on its own with the SDK's csc
// (-langversion:preview).  "Expected" lines show the diagnostics actually reported.
// =======================================================================================
//
// -- Names are real names, so duplicates are errors --------------------------------------
//   int _ = 1; int _ = 2;                          error CS0128 (duplicate local)
//   var _ = 1; var _ = 2;                          error CS0128   <- `var _` in a STATEMENT is a named local
//   using var _ = ...; using var _ = ...;          error CS0128
//   foreach (var _ in a) foreach (var _ in b) {}   error CS0136 (nested scope reuse)
//   static void M(int _, int _) {}                 error CS0100 (method params are never discards)
//   void F(int _, int _) {}   // local function    error CS0100
//   class C(int _, int _) {}  / record R(int _, int _);   error CS0100 (+ CS0102 for the record)
//   from _ in xs from _ in ys select 0;            error CS1930 (range variables are names)
//   int _ = 1; int _ = 2  ordering: `_ = x; int _ = 2;`   error CS0841 (use before declaration)
//
// -- `_` as a plain simple name when NOTHING named `_` is in scope -----------------------
//   var s = _;                                     error CS0103
//   var s = nameof(_);                             error CS0103
//   var s = $"{_}";                                error CS0103
//   _ += 1;                                        error CS0103  (compound assignment is not a discard)
//   int.TryParse("1", out _); Console.WriteLine(_); error CS0103 (the discard has no name)
//   int.TryParse("1", out var _); Console.WriteLine(_);   error CS0103
//   int.TryParse("1", out int _); Console.WriteLine(_);   error CS0103
//   var (_, x) = (1, 2); Console.WriteLine(_);     error CS0103
//   if (o is int _) Console.WriteLine(_);          error CS0103
//   if (o is var _) Console.WriteLine(_);          error CS0103
//   Func<int,int,int> f = (_, _) => _;             error CS0103 (discards introduce no name)
//   System.Func<int,int,int> f = (@_, @_) => 0;    error CS0100
//   void _() {} _ = 5;                             error CS1656 ('_' is a method group)
//
// -- Lambda mixing `_` and `@_` ----------------------------------------------------------
//   Func<int,int,int> f = (_, @_) => @_;           error CS0100 + CS0229   (`@_` is not counted as an underscore)
//   var f = _ => 1;                                error CS8917 (delegate type cannot be inferred; unrelated, but shows `_` is a name)
//
// -- `foreach` deconstruction without any `_` in scope -----------------------------------
//   int x; foreach ((_, x) in new[]{(1,2)}) {}     error CS8186 (foreach must declare its iteration variables)
//   int _ = 0; int a; foreach ((_, a) in new[]{(1,2)}) {}   error CS8186 (also with a local `_` in scope)
//
// -- `is _` / `case _:` -------------------------------------------------------------------
//   object o = 1; var b = o is _;                  error CS0246 (binds `_` as a TYPE; nothing named `_`)
//   object o = 1; int _ = 0; var b = o is _;       error CS0118 ('_' is a variable but is used like a type)
//   const int _ = 1; int i = 1; var b = i is _;    error CS0118  (local constant)
//   class C { const int _ = 1; bool M(int i) => i is _; }   error CS0246
//        NOTE: the standard (patterns.md §11.2.7) says an accessible constant `_` makes `is _` a constant pattern.
//        Roslyn does NOT do that for `is _`: the right operand of `is` is bound as a type first and the
//        constant-pattern fallback is explicitly skipped when the identifier is `_` (Binder_Operators.cs: `wasUnderscore`).
//   class C { int _ = 1; bool M(int i) => i is _; } error CS0246 (field)
//   int i = 1; switch (i) { case _: break; }       error CS0103 (switch-statement `case _` is not a discard)
//   int i = 1; var b = i is not _;                  (parses as discard pattern: `not _` => error CS8518 "can never match")
//
// -- A type named `_` in pattern designations (does not parse) ---------------------------
//   class _ {}  o is _ x                           error CS1073 / CS1002   (`_ x` is not a declaration pattern)
//   class _ {}  o is _ _                           parse errors
//   class _ {}  case _ x:                          error CS1003
//   class _ {}  o switch { _ x => 1 }              error CS1003
//   class _ {}  t is (_ a, _ b)                    error CS1003
//   class _ {}  d is { P: _ x }                    error CS1003
//        NOTE: the standard's note in §11.2.7 says "`_` may appear as the type of a declaration_pattern (e.g. `_ x`)";
//        Roslyn's parser (LanguageParser_Patterns.cs) never treats the token `_` as the start of a type there.
//        `@_ x` works (see case 7e).
//   class _ {}  o is not _                          error CS8518 (discard pattern)
//
// -- `@_` --------------------------------------------------------------------------------
//   int i = 1; var r = i switch { @_ => "a", var _ => "b" };   error CS0103 (`@_` is a simple name, nothing named `_`)
//   record R(int _);  works; record R(int _, int _);          error CS0100
//
// -- Notes -------------------------------------------------------------------------------
//   * `using _ = System.Console;` (case 1o) is a compile-time alias named `_`; it is a namespace-level
//     name so it lives in a different declaration space from locals.
//   * Roslyn gives WRN_IsTypeNamedUnderscore (CS8513) for `o is _` when `_` is a type, and
//     WRN_CaseConstantNamedUnderscore (CS8512) for `case _:` when `_` is a constant. These are the only
//     diagnostics Roslyn gives today that warn about the `_` ambiguity.
//
// -- Where the current standard text (patterns.md §11.2.7) and Roslyn disagree (observed above) --
//   * Constant `_` in a switch EXPRESSION arm or in a subpattern: standard => constant pattern; Roslyn => discard (8b, 8e-8h).
//   * Constant `_` in `x is _`: standard => constant pattern; Roslyn => error CS0246/CS0118 (the RHS is bound as a type).
//   * Constant `_` in switch-statement `case _:`: constant pattern + warning CS8512 (agrees; case 8d).
//   * Type `_` in `x is _`: type test + warning CS8513 (agrees; case 7a).
//   * Type `_` in a switch-expression arm / subpattern / `case _:`: expression arm and property subpattern are DISCARD (7b, 7d),
//     the switch-statement `case _:` is the TYPE pattern (7c).
//   * Type `_` as declaration_pattern type (`_ x`): standard says allowed; Roslyn's parser rejects it; use `@_ x` (7e).
```

* `new-behavior-tests.cs` — for each of those cases shows what the proposal does (a runnable "migrated" form wherever the behavior can be expressed today, since `@_` is available today; a clearly marked "describe-only" note where the new behavior cannot be executed because the proposal is not implemented).

```csharp
// Phase 3: how each case from existing-tests.cs behaves under the "unconditional discard token" proposal.
// Run with:  dotnet run new-behavior-tests.cs
//
// The proposal is NOT implemented, so this file can only execute what today's compiler can express.
// Each case below is therefore one of:
//
//   [SAME]      Behavior is unchanged by the proposal. Executed here with the code exactly as in existing-tests.cs.
//   [MIGRATED]  Today's meaning of `_` is a NAME, which the proposal turns into an error (or into a discard).
//               The code shown is the form that keeps the name: the identifier `@_`.  `@_` is accepted by today's
//               compiler and, under the proposal, it is the (only) spelling of an identifier named `_`, so the
//               executed output is the proposal's behavior for the migrated program.
//   [DISCARD]   Code that is valid under the proposal and, with today's compiler, means something different
//               or does not compile. These snippets are inside `#if PROPOSAL_ONLY` blocks (never compiled)
//               and are DESCRIBE-ONLY; the comment states the proposed meaning and today's behavior.
//   [ERROR]     Code that becomes an error under the proposal (DESCRIBE-ONLY: today it compiles), with the
//               migrated fix executed in the same case.
//
// The line "  BEFORE:" in a comment is the observation recorded in existing-tests.cs for the same case id.

#:property AllowUnsafeBlocks=true
#pragma warning disable CS8512, CS8513, CS0219, CS0168, CS8321, CS0414, CS0649, CS0169, CS0162, CS8794, CS8632, CS8520

using System;
using System.Collections.Generic;
using System.Linq;

Console.WriteLine("=== 1. Declarations ===");
Declarations.Run();
Console.WriteLine("\n=== 2. Lambdas ===");
Lambdas.Run();
Console.WriteLine("\n=== 3. out arguments ===");
OutArguments.Run();
Console.WriteLine("\n=== 4. Assignment and deconstruction ===");
Assignments.Run();
Console.WriteLine("\n=== 5. Patterns ===");
Patterns.Run();
Console.WriteLine("\n=== 6. nameof / interpolation / member access ===");
NamesAndStrings.Run();
Console.WriteLine("\n=== 7. A type named _ ===");
TypeNamedUnderscore.Run();
Console.WriteLine("\n=== 8. A constant named _ ===");
ConstNamedUnderscore.Run();
Console.WriteLine("\n=== 9. Members named _ ===");
MemberNamedUnderscore.Run();
Console.WriteLine("\n=== 10. @_ under the proposal ===");
AtUnderscore.Run();
Console.WriteLine("\n=== 11. A TYPE named _ of every kind (migrated to @_; see the Option X / Option Y comment block at the end of section 11) ===");
TypeKinds.Run();

static class TypeOut
{
    public static void Show(string label, object? value) => Console.WriteLine($"{label,-72}: {value}");
}

static class Out
{
    public static void Show(string label, string kind, object? value) => Console.WriteLine($"{label,-6}{kind,-12}{value}");
}

static class Declarations
{
    public static void Run()
    {
        // 1a  BEFORE: `int _ = 1; ... _` => named local.
        //     AFTER : `int _ = 1;` is an error at the declaration, even if never read.  Migrated:
        { int @_ = 1; Out.Show("1a", "[MIGRATED]", $"`int @_ = 1;` named local, value {@_}"); }
#if PROPOSAL_ONLY
        { int _ = 1; Console.WriteLine(_); }   // [ERROR] local declarations require an identifier; use '@_'
        { int _ = Compute(); }                // [ERROR] even an unused local requires an identifier; use `_ = Compute();` to discard
#endif

        // 1b  BEFORE: `var _ = 2;` => named local.  AFTER: error at the declaration.  Migrated:
        { var @_ = 2; Out.Show("1b", "[MIGRATED]", $"`var @_ = 2;` named local, value {@_}"); }
#if PROPOSAL_ONLY
        { var _ = 2; var _ = 3; }              // [ERROR] both declarations are invalid (today: CS0128 for the duplicate)
#endif

        // 1c  BEFORE: `int _ = 1; _ = 3;` assigned the local.  AFTER: the second statement is a discard assignment;
        //     the first is an error at the declaration.  Migrated:
        { int @_ = 1; @_ = 3; Out.Show("1c", "[MIGRATED]", $"`int @_ = 1; @_ = 3;` now {@_}"); }

        // 1d  foreach
        {
            int sum = 0;
            foreach (var @_ in new[] { 10, 20 }) sum += @_;
            Out.Show("1d", "[MIGRATED]", $"`foreach (var @_ ...) sum += @_;` sum {sum}");
        }
#if PROPOSAL_ONLY
        foreach (var _ in new[] { 10, 20 }) Console.WriteLine("tick");   // [ERROR] iteration-variable declarations require an identifier
        foreach (var _ in a) foreach (var _ in b) { }                   // [ERROR] both declarations are invalid (today: CS0136)
        foreach (var _ in new[] { 10, 20 }) sum += _;                   // [ERROR] invalid iteration-variable declaration and read
#endif

        // 1e  for
        {
            int sum = 0;
            for (int @_ = 0; @_ < 3; @_++) sum += @_;
            Out.Show("1e", "[MIGRATED]", $"`for (int @_ = 0; ...)` sum {sum}");
        }

        // 1f  using var
        {
            using var @_ = new System.IO.MemoryStream(new byte[4]);
            Out.Show("1f", "[MIGRATED]", $"`using var @_ = ...; @_.Length` = {@_.Length}");
        }
#if PROPOSAL_ONLY
        using var _ = new System.IO.MemoryStream();  // [ERROR] resource declarations require an identifier; use '@_'
        using var _ = new System.IO.MemoryStream();  // [ERROR] still invalid (today: CS0128 for the duplicate)
#endif

        // 1g  catch
        {
            try { throw new InvalidOperationException("boom"); }
            catch (Exception @_) { Out.Show("1g", "[MIGRATED]", $"`catch (Exception @_)` => {@_.Message}"); }
        }
#if PROPOSAL_ONLY
        try { } catch (Exception _) { }   // [ERROR] use `catch (Exception)` or a named variable `catch (Exception @_)`
#endif

        // 1h  method parameter: BEFORE named; AFTER error at the declaration (not a discard position).  Migrated:
        Out.Show("1h", "[MIGRATED]", MethodParam(41));
#if PROPOSAL_ONLY
        static string MethodParamBad(int _) => "";   // [ERROR] '_' is a discard; use '@_' to name a parameter '_'
#endif

        // 1i  local function named `_`: AFTER error at the declaration.  Migrated:
        {
            int @_() => 5;
            Out.Show("1i", "[MIGRATED]", $"local function `@_`, call `@_()` = {@_()}");
        }

        // 1j / 1k  query variables
        {
            var q = from @_ in new[] { 1, 2, 3 } select @_ * 2;
            Out.Show("1j", "[MIGRATED]", string.Join(",", q));
        }
        {
            var q = from x in new[] { 1, 2, 3 } let @_ = x * 10 select @_;
            Out.Show("1k", "[MIGRATED]", string.Join(",", q));
        }

        // 1l  tuple element name
        {
            var t = (@_: 1, b: 2);
            Out.Show("1l", "[MIGRATED]", $"`(@_: 1, b: 2)`, `t.@_` = {t.@_}");
        }

        // 1m  named argument
        Out.Show("1m", "[MIGRATED]", NamedArg(@_: 7));

        // 1n  label
        {
            goto @_;
        @_:
            Out.Show("1n", "[MIGRATED]", "`goto @_; @_:` works");
        }
    }

    static string MethodParam(int @_) => $"parameter `@_`, value {@_}";
    static string NamedArg(int @_) => $"parameter `@_`, value {@_}";
    static int Compute() => 0;
}

static class Lambdas
{
    public static void Run()
    {
        // 2a-2e: today a single `_` is a NAME.  AFTER: always a discard parameter; reading it is an error.
        Func<int, int> one = @_ => @_ + 1;
        Out.Show("2a", "[MIGRATED]", $"`@_ => @_ + 1`  f(1) = {one(1)}");
        Func<int, int> oneParen = (@_) => @_ + 1;
        Out.Show("2b", "[MIGRATED]", $"`(@_) => @_ + 1`  f(1) = {oneParen(1)}");
        Func<int, int> oneTyped = (int @_) => @_ + 1;
        Out.Show("2c", "[MIGRATED]", $"`(int @_) => @_ + 1`  f(1) = {oneTyped(1)}");
        Func<int, int> anon = delegate (int @_) { return @_ + 1; };
        Out.Show("2d", "[MIGRATED]", $"`delegate (int @_) {{ ... }}`  f(1) = {anon(1)}");
        Func<int, int, int> mixed = (@_, x) => @_ + x;
        Out.Show("2e", "[MIGRATED]", $"`(@_, x) => @_ + x`  f(1,2) = {mixed(1, 2)}");
#if PROPOSAL_ONLY
        Func<int, int> d1 = _ => 0;                       // [DISCARD] discard parameter (today: a named parameter)
        Func<int, int> e1 = _ => _ + 1;                   // [ERROR] '_' is a discard
        Func<int, int, int> e2 = (_, x) => _ + x;         // [ERROR]
        Func<int, int, int> m = (_, @_) => @_;            // [DISCARD+NAMED] OK: first is a discard, second is a parameter named '_'. Today CS0100/CS0229
        Func<int, string> r = _ => "const";               // [DISCARD] no name introduced; no conflict with an outer `_`
#endif

        // 2f-2i: already discards today and stay discards.  [SAME]
        Func<int, int, int> two = (_, _) => 0;
        Out.Show("2f", "[SAME]", "`(_, _) => 0` discards");
        Func<int, int, int, int> three = (_, x, _) => x;
        Out.Show("2g", "[SAME]", $"`(_, x, _) => x` -> {three(1, 2, 3)}");
        Func<int, int, int> typedTwo = (int _, int _) => 0;
        Out.Show("2h", "[SAME]", "`(int _, int _) => 0` discards");
        Func<int, int, int> anonTwo = delegate (int _, int _) { return 0; };
        Out.Show("2i", "[SAME]", "`delegate (int _, int _)` discards");

        // 2j  BEFORE: `string _ = ..; (_, _) => _` bound the body `_` to the OUTER local.
        //     AFTER : the outer declaration `string _` is an error; the lambda body `_` is also an invalid discard read.
        //     Migrated: outer local is `@_`; the body must say `@_`.
        string @_ = "outer-local";
        Func<int, int, string> discardsSeeOuter = (_, _) => @_;
        Out.Show("2j", "[MIGRATED]", $"`(_, _) => @_` sees outer `@_`: \"{discardsSeeOuter(1, 2)}\"");

        // 2k  BEFORE: `_ => _.ToString()` parameter hid the outer `_`.  Migrated to a named parameter `@_` (which hides the outer `@_`)
        Func<int, string> singleHidesOuter = @_ => @_.ToString();
        Out.Show("2k", "[MIGRATED]", $"`@_ => @_.ToString()` parameter hides outer `@_`: \"{singleHidesOuter(5)}\"");

        Func<int, int, int> nestedLocal = (_, _) => { int @x = 3; return x; };
        Out.Show("2l", "[SAME]", "discard params don't conflict with a body local; the local is just not named `_` any more");

        Func<int, int, bool> outInBody = (_, _) => int.TryParse("12", out var _);
        Out.Show("2m", "[SAME]", $"`out var _` inside a discard lambda: {outInBody(0, 0)}");

        Action<int> assignInBody = @_ => { @_ = 5; Out.Show("2n", "[MIGRATED]", $"assigns to the named parameter `@_` (now {@_})"); };
        assignInBody(1);
#if PROPOSAL_ONLY
        Action<int> assignDiscard = _ => { _ = 5; };      // [DISCARD] discard assignment inside; no parameter is written
#endif

        Func<int, Func<int, int, int>> nested = _ => (_, _) => 0;
        Out.Show("2o", "[SAME]", "inner `(_, _)` discards; outer `_` is a discard parameter under the proposal (a named parameter today), still legal");

        var withDefaults = (int _ = 1, int _ = 2) => 0;
        Out.Show("2p", "[SAME]", "`(int _ = 1, int _ = 2) => 0` discards");
    }
}

static class OutArguments
{
    public static void Run()
    {
        // 3a  [SAME]
        { int.TryParse("5", out _); Out.Show("3a", "[SAME]", "`out _` discard"); }

        // 3b  BEFORE: with a local `_`, `out _` assigned the local.  AFTER: always a discard.
        //     To keep the old behavior the local is named `@_` and the argument must be `out @_`.
        {
            int @_ = -1;
            int.TryParse("5", out @_);
            Out.Show("3b", "[MIGRATED]", $"`int @_ = -1; out @_` assigns the named local: now {@_}");
        }
#if PROPOSAL_ONLY
        {
            int @_ = -1;
            int.TryParse("5", out _);                     // [DISCARD] discard; `@_` stays -1  (today: assigns the local named `_`/`@_`)
        }
#endif

        // 3c / 3d / 3e  [SAME]: `out var _` / `out int _` are discards today and under the proposal
        { int @_ = -1; int.TryParse("5", out var _); Out.Show("3c", "[SAME]", $"`out var _`: local `@_` unchanged = {@_}"); }
        { int @_ = -1; int.TryParse("5", out int _); Out.Show("3d", "[SAME]", $"`out int _`: local `@_` unchanged = {@_}"); }
        { string @_ = "text"; int.TryParse("5", out var _); Out.Show("3e", "[SAME]", "`out var _` with a string `@_` in scope"); }
#if PROPOSAL_ONLY
        { string @_ = "text"; int.TryParse("5", out _); } // [DISCARD] OK now (today: CS1503 because `_` bound to the string)
#endif
    }
}

static class Assignments
{
    public static void Run()
    {
        // 4a  [SAME]
        { _ = 5; Out.Show("4a", "[SAME]", "`_ = 5;` discard assignment"); }

        // 4b  BEFORE: `int _ = 0; _ = 5;` assigned the local.  Migrated:
        { int @_ = 0; @_ = 5; Out.Show("4b", "[MIGRATED]", $"`int @_ = 0; @_ = 5;` => {@_}"); }
#if PROPOSAL_ONLY
        { int @_ = 0; _ = 5; }                            // [DISCARD] discard assignment; `@_` stays 0 (today: assigns it)
#endif

        // 4c  [SAME]
        { int x; (_, x) = (1, 2); Out.Show("4c", "[SAME]", $"`(_, x) = (1, 2);` x = {x}"); }

        // 4d  BEFORE: `int _ = 0; (_, x) = (1, 2);` assigned the local `_` = 1.  Migrated:
        { int @_ = 0; int x; (@_, x) = (1, 2); Out.Show("4d", "[MIGRATED]", $"`(@_, x) = (1, 2)` => `@_` = {@_}, x = {x}"); }
#if PROPOSAL_ONLY
        { int @_ = 0; int x; (_, x) = (1, 2); }           // [DISCARD] first element discarded; `@_` stays 0 (today: assigns it)
#endif

        // 4e / 4f  [SAME]
        { int @_ = 0; var (_, y) = (1, 2); Out.Show("4e", "[SAME]", $"`var (_, y)`: `@_` still {@_}, y = {y}"); }
        { int @_ = 0; (int _, int z) = (1, 2); Out.Show("4f", "[SAME]", $"`(int _, int z)`: `@_` still {@_}, z = {z}"); }

        // 4g  BEFORE: bare `_` in the deconstruction assigned the local.  Migrated (explicit `@_`):
        { int @_ = 0; (int a, var _, @_) = (1, 2, 3); Out.Show("4g", "[MIGRATED]", $"`(int a, var _, @_)`: `@_` = {@_}, a = {a}"); }

        // 4h  [SAME]
        { int x = 0; _ = ref x; Out.Show("4h", "[SAME]", "`_ = ref x;` ref discard assignment"); }

        // 4i  BEFORE: `ref var _ = ref x;` => named ref local.  AFTER: error (ref discard declarations are not part of the proposal; use `_ = ref x;`)
        { int x = 0; ref var @_ = ref x; @_ = 9; Out.Show("4i", "[MIGRATED]", $"`ref var @_ = ref x;` writes through: x = {x}"); }

        // 4j  BEFORE: `_ += 4` compound assignment on the named local.  AFTER: error for a discard.  Migrated:
        { int @_ = 0; @_ += 4; Out.Show("4j", "[MIGRATED]", $"`@_ += 4` => {@_}"); }
#if PROPOSAL_ONLY
        { _ += 4; }                                       // [ERROR] compound assignment to a discard (already an error today when `_` is not in scope: CS0103)
#endif

        // 4k  [SAME]
        foreach (var (_, b) in new List<(int, int)> { (1, 2) }) Out.Show("4k", "[SAME]", $"`foreach (var (_, b) ...)` b = {b}");

        // 4l  foreach ((_, a) in ...) is error CS8186 today and under the proposal (needs declarations).  [SAME, describe-only]
    }
}

static class Patterns
{
    public static void Run()
    {
        object o = 42;
        Out.Show("5a", "[SAME]", $"`o is int _` -> {o is int _}");
        { int @_ = 0; Out.Show("5b", "[SAME]", $"`o is int _` with `@_` in scope -> {o is int _}"); }
        Out.Show("5c", "[SAME]", $"`o is var _` -> {o is var _}");
        if (o is int n && n > 0) Out.Show("5d", "[SAME]", "named designation");
        Out.Show("5e", "[SAME]", $"`t is (_, _)` -> {(1, 2) is (_, _)}");
        Out.Show("5f", "[SAME]", $"`s is {{ Length: _ }}` -> {"abc" is { Length: _ }}");
        int[] arr = { 1, 2 };
        Out.Show("5g", "[SAME]", $"`a is [_, ..]` -> {arr is [_, ..]}");
        {
            int i = 5;
            Out.Show("5h", "[SAME]", $"`i switch {{ 1 => \"one\", _ => \"other\" }}` -> {i switch { 1 => "one", _ => "other" }}");
        }
        {
            int @_ = 1;
            int i = 5;
            Out.Show("5i", "[SAME]", $"`_ =>` with `@_` in scope is still the discard -> {i switch { 1 => "one", _ => "other" }}");
        }
        { int i = 5; switch (i) { case var _: Out.Show("5j", "[SAME]", "`case var _:` matches everything"); break; } }
        { int i = 5; switch (i) { case int _: Out.Show("5k", "[SAME]", "`case int _:`"); break; } }
        { int @_ = 7; Out.Show("5l", "[SAME]", $"`(1,2) is (_, _)` with `@_` in scope -> {(1, 2) is (_, _)}"); }

        // 5m  BEFORE: `t is (_: 1, b: 2)` used a tuple element NAME `_`.  AFTER: error.  Migrated:
        {
            var t = (@_: 1, b: 2);
            Out.Show("5m", "[MIGRATED]", $"`t is (@_: 1, b: 2)` -> {t is (@_: 1, b: 2)}");
        }
#if PROPOSAL_ONLY
        { var t = (_: 1, b: 2); var r = t is (_: 1, b: 2); }   // [ERROR] '_' is a discard: tuple element names and subpattern labels need '@_'
        { object p = 1; var r = p is _; }                      // [ERROR] bare '_' as the whole `is` pattern (unchanged rule; use `var _`)
        { int i = 1; switch (i) { case _: break; } }           // [ERROR] bare '_' as the whole `case` pattern (use `case var _:`)
#endif
    }
}

static class NamesAndStrings
{
    public static void Run()
    {
        // 6a/6b  BEFORE: `nameof(_)` and `$"{_}"` with a local `_`.  AFTER: errors; migrated with `@_`.
        {
            int @_ = 3;
            Out.Show("6a", "[MIGRATED]", $"`nameof(@_)` = {nameof(@_)}");
            Out.Show("6b", "[MIGRATED]", $"`$\"{{@_}}\"` = {@_}");
        }
        // 6c  tuple element access `t._` => `t.@_`
        { var t = (@_: "item", b: 2); Out.Show("6c", "[MIGRATED]", $"`t.@_` = {t.@_}"); }
#if PROPOSAL_ONLY
        { int x = 0; var s = nameof(_); var u = $"{_}"; var v = _; }   // [ERROR] x3: a discard cannot be read or named
#endif
    }
}

static class TypeNamedUnderscore
{
    public static void Run()
    {
        T1.X.Run(); T2.X.Run(); T3.X.Run(); T4.X.Run(); T5.X.Run(); T6.X.Run();
    }
}

// Every `class _` below is spelled `@_` under the proposal (declaring a type with the bare `_` is an error).
namespace T1
{
    class @_ { }
    static class X
    {
        public static void Run()
        {
            object o = new @_();
            object p = "not an _";
            // 7a  BEFORE: `o is _` was an is-type test (warning CS8513).  AFTER: `o is _` is an error; the type test is `o is @_`.
            Out.Show("7a", "[MIGRATED]", $"`o is @_` -> {o is @_}, string -> {p is @_}");
        }
    }
}
namespace T2
{
    class @_ { }
    static class X
    {
        public static void Run()
        {
            object p = "not an _";
            var r = p switch { _ => "arm `_` matched" };
            Out.Show("7b", "[SAME]", $"{r}  (the `_` arm is the discard today and under the proposal)");
        }
    }
}
namespace T3
{
    class @_ { }
    static class X
    {
        public static void Run()
        {
            object p = new @_();
            string r = "no match";
            // 7c  BEFORE: `case _:` with a type `_` was the TYPE pattern.  AFTER: bare `case _:` is an error; write `case @_:` for the type pattern
            switch (p) { case @_: r = "case @_: matched the type"; break; }
            Out.Show("7c", "[MIGRATED]", r);
#if PROPOSAL_ONLY
            switch (p) { case _: break; }                  // [ERROR] use `case var _:` for "match anything"
#endif
        }
    }
}
namespace T4
{
    class @_ { }
    class D { public object P = "str"; }
    static class X
    {
        public static void Run()
        {
            var d = new D();
            Out.Show("7d", "[SAME]", $"`{{ P: _ }}` on a string is still a discard subpattern -> {(d is { P: _ })}");
        }
    }
}
namespace T5
{
    class @_ { public override string ToString() => "an _"; }
    static class X
    {
        public static void Run()
        {
            object o = new @_();
            if (o is @_ x) Out.Show("7e", "[SAME]", $"`o is @_ x` -> {x}  (the only spelling that parses today, unchanged)");
            if (o is @_) Out.Show("7f", "[SAME]", "`o is @_` type test");
            var r = o switch { @_ => "type-`_` pattern", _ => "discard" };
            Out.Show("7g", "[SAME]", r);
        }
    }
}
namespace T6
{
    class @_ { public string Hi() => "hi from type _"; }
    static class X
    {
        public static void Run()
        {
            @_ v = new @_();
            Out.Show("7h", "[MIGRATED]", $"`@_ v = new @_();` -> {v.Hi()}");
            var list = new List<@_> { new @_() };
            Out.Show("7i", "[MIGRATED]", $"`List<@_>` -> Count {list.Count}");
            int.TryParse("1", out var _);
            Out.Show("7j", "[SAME]", "`out var _` is a discard");
        }
    }
}

static class ConstNamedUnderscore
{
    const int @_ = 1;

    public static void Run()
    {
        // 8a/8b  BEFORE: in a switch EXPRESSION the arm `_` stayed a discard even with a constant `_` in scope.  [SAME]
        Out.Show("8a", "[SAME]", $"`5 switch {{ 2 => .., _ => \"discard arm\" }}` -> {5 switch { 2 => "two", _ => "discard arm" }}");
        // 8c  `@_ =>` is the constant pattern
        Out.Show("8c", "[SAME]", $"`5 switch {{ @_ => \"const\", var _ => \"discard\" }}` -> {5 switch { @_ => "const", var _ => "discard" }}");
        // 8d  BEFORE: switch STATEMENT `case _:` was the constant pattern (CS8512).  AFTER: error; write `case @_:`.
        string r = "none";
        switch (5) { case @_: r = "matched constant"; break; default: r = "`case @_:` did NOT match 5 (constant is 1)"; break; }
        Out.Show("8d", "[MIGRATED]", r);
        // 8e-8h  subpatterns: discard before and after.  [SAME]
        Out.Show("8e", "[SAME]", $"`(5, 6) is (_, _)` -> {(5, 6) is (_, _)}");
        Out.Show("8g", "[SAME]", $"`\"s\" is {{ Length: _ }}` -> {"s" is { Length: _ }}");
#if PROPOSAL_ONLY
        const int _ = 1;                                   // [ERROR] a constant cannot be declared with a discard-token; use `const int @_ = 1;`
        switch (5) { case _: break; }                      // [ERROR] bare '_' as a whole `case` pattern
#endif
    }
}

static class MemberNamedUnderscore
{
    public static void Run()
    {
        var c = new HasFieldUnderscore();
        // 9a-9c  BEFORE: inside the class, `_ = 5`, `out _`, `(_, x) = ..` assigned the FIELD named `_`.
        //        AFTER : the field must be declared `@_`, and those statements become DISCARDS (they no longer touch the field) - this is
        //                the "silent change" described in the proposal (section D) when the field comes from another assembly.
        //                The migrated form that keeps the old behavior writes `@_` at each use.
        Out.Show("9a", "[MIGRATED]", c.AssignViaSimpleName());
        Out.Show("9b", "[MIGRATED]", c.OutViaSimpleName());
        Out.Show("9c", "[MIGRATED]", c.DeconstructViaSimpleName());
        Out.Show("9d", "[SAME]", c.SwitchArm());
        Out.Show("9e", "[MIGRATED]", new HasMethodUnderscore().Call());
        Out.Show("9f", "[MIGRATED]", MyEnum.@_);
        Out.Show("9g", "[MIGRATED]", new Rec(3).@_);
        Out.Show("9h", "[MIGRATED]", new Prim(4).Get());
        Out.Show("9i", "[MIGRATED]", new HasPublicUnderscore { @_ = 6 }.@_);
        Out.Show("9j", "[MIGRATED]", new HasPublicUnderscore { @_ = 6 } is { @_: 6 });
    }

    class HasFieldUnderscore
    {
        int @_;
        public string AssignViaSimpleName() { @_ = 5; return $"`@_ = 5` assigned the FIELD: {@_}"; }
        public string OutViaSimpleName() { int.TryParse("8", out @_); return $"`out @_` wrote the FIELD: {@_}"; }
        public string DeconstructViaSimpleName() { int x; (@_, x) = (11, 2); return $"`(@_, x) = ..` wrote the FIELD: {@_}"; }
        public string SwitchArm() => 99 switch { 1 => "one", _ => "discard arm (unchanged)" };
#if PROPOSAL_ONLY
        public void Discards() { _ = 5; int.TryParse("8", out _); int x; (_, x) = (1, 2); }   // [DISCARD] none of these touch the field `@_`
#endif
    }
    class HasMethodUnderscore { void @_() { } public string Call() { @_(); return "called method `@_()`"; } }
    enum MyEnum { @_ }
    record Rec(int @_);
    class Prim(int @_) { public int Get() => @_; }
    class HasPublicUnderscore { public int @_; }
}

static class AtUnderscore
{
    public static void Run()
    {
        // Today `@_` and `_` are the same name.  Under the proposal they are different:
        //   `@_` is an identifier named "_", `_` is the discard token. This section only exercises what `@_` can do (all of it works today).
        { int @_ = 5; Out.Show("10a", "[SAME]", $"`int @_ = 5; @_` -> {@_}"); }
        { Func<int, int> f = @_ => @_ + 1; Out.Show("10b", "[SAME]", $"`@_ => @_ + 1` -> {f(1)}"); }
        { int @_ = 1; int.TryParse("9", out @_); Out.Show("10c", "[SAME]", $"`out @_` -> {@_}"); }
        { Out.Show("10d", "[DESCRIBE]", "`(_, @_) => @_`: today CS0100/CS0229; under the proposal OK (first a discard, second a parameter named `_`)"); }
#if PROPOSAL_ONLY
        { int @_ = 7; Console.WriteLine(_); }              // [ERROR] bare `_` does not refer to `@_`
        { Func<int, int, int> f = (_, @_) => @_; }         // OK under the proposal
#endif
    }
}
// ---- Section 11 body: type `_` of every kind, migrated to `@_` ----
static class TypeKinds { public static void Run() {
 K1.X.Run(); K2.X.Run(); K3.X.Run(); K4.X.Run(); K5.X.Run(); K6.X.Run(); K7.X.Run();
 K8.X.Run(); K9.X.Run(); K10.X.Run(); K11.X.Run(); K12.X.Run(); K13.X.Run(); K14.X.Run(); K15.X.Run(); } }
// 11A: class `_`
namespace K1
{
    class @_ : Exception, IDisposable
    {
        public static int Foo = 7;
        public int P = 3;
        public void Dispose() { }
    }

    static class X
    {
        static void Q(out @_ p) => p = new @_();

        public static void Run()
        {
            @_ x = new @_();
            object o = x, s = "a string";
            TypeOut.Show("[MIGRATED] 11A-1 class: decl / cast / new / typeof / nameof", $"`@_ x`={x.P}; `(@_)o`={((@_)o).P}; `typeof(@_)`={typeof(@_).Name}; `nameof(@_)`={nameof(@_)}; `@_.Foo`={@_.Foo}");
            TypeOut.Show("[MIGRATED] 11A-2 class: generic args / arrays / default", $"`List<@_>`={new List<@_> { x }.Count}; `new @_[2]`={new @_[2].Length}; `default(@_)` is null={default(@_) is null}");
            TypeOut.Show("[MIGRATED] 11A-3 class: `is @_` (string, `@_` instance)", $"{s is @_}, {o is @_}  (type test)");
            TypeOut.Show("[SAME] 11A-4 class: `is _ or int` / `is (_)` (string)", $"{s is _ or int}, {s is (_)}  (true => `_` is the DISCARD here, not the type)");
            TypeOut.Show("[SAME] 11A-5 class: `s is { } and _` / `is int or _`", $"{s is { } and _}, {s is int or _}  (true => discard)");
            string r = "no match";
            switch (s) { case @_: r = "matched"; break; }
            TypeOut.Show("[MIGRATED] 11A-6 class: statement `case @_:` (string)", $"{r}  (not matched => TYPE pattern)");
            TypeOut.Show("[SAME] 11A-7 class: switch expression arm `_ =>` (string)", s switch { _ => "matched" } + "  (matched => DISCARD)");
            TypeOut.Show("[SAME] 11A-8 class: positional/property subpattern `_`", $"{(s, 1) is (_, _)}, {(P: s, Q: 1) is { P: _ }}  (discard subpatterns)");
            Func<@_, @_, int> f = (@_ a, @_ b) => a.P + b.P;
            Func<@_, int> g = @_ => @_.P;
            TypeOut.Show("[MIGRATED] 11A-9 class: lambda `(@_ a, @_ b)` / `@_ => @_.P`", $"{f(x, x)}, {g(x)}  (the lambda parameter `@_` hides the type inside the body)");
            (@_, @_) pair = (x, x); (@_ a, @_ b) = pair;
            TypeOut.Show("[MIGRATED] 11A-10 class: deconstruction `(@_ a, @_ b) = (x, x)`", $"{a.P + b.P}");
            @_ q; Q(out q);
            Q(out @_ q2);
            TypeOut.Show("[MIGRATED] 11A-11 class: `Q(out @_ q)` typed out declaration", $"{q.P}, {q2.P}");
            int n = 0; foreach (@_ e in new[] { x, x }) n += e.P;
            TypeOut.Show("[MIGRATED] 11A-12 class: `foreach (@_ e in ...)`", n);
            try { throw new @_(); } catch (@_ e) { TypeOut.Show("[MIGRATED] 11A-13 class: `catch (@_ e)`", e.GetType().Name); }
            try { throw new @_(); } catch (@_) { TypeOut.Show("[MIGRATED] 11A-14 class: `catch (@_)`", "caught"); }
            using (@_ u = new @_()) { TypeOut.Show("[MIGRATED] 11A-15 class: `using (@_ u = new @_())`", "ok"); }
            {
                @_ @_ = new @_();
                TypeOut.Show("[MIGRATED] 11A-16 class: Color Color `@_ @_ = new @_();` `@_.Foo`, `@_.P`", $"{@_.Foo}, {@_.P}");
            }
            @_ _x = new @_(); @_ x2 = new(); var xs = new @_[] { new @_() };
            TypeOut.Show("[MIGRATED] 11A-17 class: `@_ _x`, target-typed `new()`, `new @_[] {..}`", $"{_x.P}, {x2.P}, {xs.Length}");
            TypeOut.Show("[MIGRATED] 11A-18 class: `o as @_`, `typeof(@_[])`", $"{(o as @_)?.P}, {typeof(@_[]).Name}");
        }
    }
}

// 11B: struct `_`
namespace K2
{
    struct @_ { public static int Foo = 7; public int P; }

    static class X
    {
        public static void Run()
        {
            @_ x = new @_ { P = 3 };
            object o = x, s = "a string";
            Span<@_> sp = stackalloc @_[2];
            @_? n = null;
            TypeOut.Show("[MIGRATED] 11B-1 struct: `@_ x`, `@_.Foo`, `Span<@_> = stackalloc @_[2]`, `@_?`", $"{x.P}, {@_.Foo}, {sp.Length}, {n.HasValue}");
            TypeOut.Show("[MIGRATED] 11B-2 struct: `o is @_`, `s is @_`, `(@_)o`, `default(@_)`", $"{o is @_}, {s is @_}, {((@_)o).P}, {default(@_).P}");
            unsafe { @_* p = &x; TypeOut.Show("[MIGRATED] 11B-3 struct: `@_* p`, `sizeof(@_)`", $"{p->P}, {sizeof(@_)}"); }
        }
    }
}

// 11C: interface `_`
namespace K3
{
    interface @_ { static int Foo = 7; int P { get; } }
    class Impl : @_ { public int P => 3; }
    class D : @_ { public int P => 4; }

    static class X
    {
        public static void Run()
        {
            @_ i = new Impl();
            object o = i, s = "a string";
            TypeOut.Show("[MIGRATED] 11C-1 interface: `@_ i`, `@_.Foo`, `o is @_`, `s is @_`, `(@_)o`", $"{i.P}, {@_.Foo}, {o is @_}, {s is @_}, {((@_)o).P}");
            TypeOut.Show("[MIGRATED] 11C-2 interface: `class D : @_`", new D().P);
        }
    }
}

// 11D: enum `_`
namespace K4
{
    enum @_ { Foo = 1, Bar = 2 }

    static class X
    {
        public static void Run()
        {
            @_ e = @_.Foo; const @_ k = @_.Bar; object o = e, s = "a string";
            var c = (@_)2;
            TypeOut.Show("[MIGRATED] 11D-1 enum: `@_ e = @_.Foo`, `const @_ k`, `(@_)2`, `e == @_.Foo`", $"{e}, {k}, {c}, {e == @_.Foo}");
            TypeOut.Show("[MIGRATED] 11D-2 enum: `o is @_`, `s is @_`, `typeof(@_)`, `nameof(@_.Foo)`", $"{o is @_}, {s is @_}, {typeof(@_).Name}, {nameof(@_.Foo)}");
            TypeOut.Show("[MIGRATED] 11D-3 enum: `Enum.GetValues<@_>()`", string.Join(",", Enum.GetValues<@_>()));
        }
    }
}

// 11E: delegate `_`
namespace K5
{
    delegate int @_(int a);

    static class X
    {
        public static void Run()
        {
            @_ d = a => a + 1; @_ d2 = new @_(a => a + 2);
            object o = d, s = "a string";
            TypeOut.Show("[MIGRATED] 11E-1 delegate: `@_ d = a => ...`, `new @_(...)`, `o is @_`, `s is @_`", $"{d(1)}, {d2(1)}, {o is @_}, {s is @_}");
            Func<int, int> f = @_ => @_ + 1;
            TypeOut.Show("[MIGRATED] 11E-2 delegate: `Func<int,int> f = @_ => @_ + 1` (lambda param hides type)", f(1));
        }
    }
}

// 11F: record `_`
namespace K6
{
    record @_(int P) { public static int Foo = 7; }

    static class X
    {
        public static void Run()
        {
            @_ x = new @_(3); object o = x, s = "a string";
            var y = x with { P = 4 };
            TypeOut.Show("[MIGRATED] 11F-1 record: `new @_(3)`, `with`, `@_.Foo`, `o is @_`, `s is @_`, `(@_)o`", $"{x.P}, {y.P}, {@_.Foo}, {o is @_}, {s is @_}, {((@_)o).P}");
            TypeOut.Show("[MIGRATED] 11F-2 record: `typeof(@_)`, `nameof(@_)`, ToString", $"{typeof(@_).Name}, {nameof(@_)}, {x}");
        }
    }
}

// 11G: record struct `_`
namespace K7
{
    record struct @_(int P) { public static int Foo = 7; }

    static class X
    {
        public static void Run()
        {
            @_ x = new @_(3); object o = x, s = "a string";
            TypeOut.Show("[MIGRATED] 11G-1 record struct: `new @_(3)`, `@_.Foo`, `o is @_`, `s is @_`, `(@_)o`", $"{x.P}, {@_.Foo}, {o is @_}, {s is @_}, {((@_)o).P}");
        }
    }
}

// 11H: generic type `_<T>`
namespace K8
{
    class @_<T> { public static int Foo = 7; public T? V; }

    static class X
    {
        public static void Run()
        {
            @_<int> x = new @_<int> { V = 3 }; object o = x, s = "a string";
            TypeOut.Show("[MIGRATED] 11H-1 generic: `@_<int> x`, `new @_<int>()`, `@_<int>.Foo`, `typeof(@_<>)`", $"{x.V}, {@_<int>.Foo}, {typeof(@_<>).Name}");
            TypeOut.Show("[MIGRATED] 11H-2 generic: `o is @_<int>`", "does not parse (CS1525); see existing-tests.cs. Using `@_<int>`: " + (o is @_<int>) + ", " + (s is @_<int>));
        }
    }
}

// 11I: using alias `_`
namespace K9
{
    using @_ = System.Text.StringBuilder;

    static class X
    {
        public static void Run()
        {
            @_ sb = new @_("ab"); object o = sb, s = "a string";
            TypeOut.Show("[MIGRATED] 11I-1 alias: `@_ sb = new @_(..)`, `o is @_`, `s is @_`, `(@_)o`, `typeof(@_)`", $"{sb.Length}, {o is @_}, {s is @_}, {((@_)o).Length}, {typeof(@_).Name}");
            TypeOut.Show("[MIGRATED] 11I-2 alias: `nameof(@_)`", nameof(@_));
        }
    }
}

// 11J: nested type `_`
namespace K10
{
    static class X
    {
        class @_ { public static int Foo = 7; public int P = 3; }

        public static void Run()
        {
            @_ x = new @_(); object o = x, s = "a string";
            TypeOut.Show("[MIGRATED] 11J-1 nested: `@_ x = new @_()`, `@_.Foo`, `o is @_`, `s is @_`, `typeof(@_).FullName`", $"{x.P}, {@_.Foo}, {o is @_}, {s is @_}, {typeof(@_).FullName}");
        }
    }
}

// 11K: type parameter `_`
namespace K11
{
    static class X
    {
        static string G<@_>(object o) where @_ : new()
        {
            @_ v = new @_(); var d = default(@_);
            return $"`@_ v = new @_()`={v}; `typeof(@_)`={typeof(@_).Name}; `o is @_`={o is @_}; `nameof(@_)`={nameof(@_)}";
        }

        static string H<@_>(object o) => $"{((@_)o)}, {o is @_}, {o is @_ or int}";

        public static void Run()
        {
            TypeOut.Show("[MIGRATED] 11K-1 type param: `G<@_>` body uses `@_` as a type", G<object>(1));
            TypeOut.Show("[MIGRATED] 11K-2 type param: `H<@_>` `(@_)o`, `o is @_`, `o is @_ or int`", H<string>("s"));
        }
    }
}

// 11L: namespace `_`
namespace K12
{
    namespace @_ { public class Foo { public static int Bar = 7; } }
    namespace Use
    {
        using @_;
        static class X
        {
            public static void Run()
            {
                var a = new @_.Foo(); var b = new Foo();
                TypeOut.Show("[MIGRATED] 11L-1 namespace: `new @_.Foo()`, `using @_;` + `Foo`, `@_.Foo.Bar`, `nameof(@_.Foo)`", $"{a.GetType().FullName}, {b.GetType().Name}, {@_.Foo.Bar}, {nameof(@_.Foo)}");
                TypeOut.Show("[MIGRATED] 11L-2 namespace: `nameof(@_)`", nameof(@_));
            }
        }
    }
    static class X { public static void Run() => Use.X.Run(); }
}

// 11M: `using static _;`
namespace K13
{
    static class @_ { public static int Foo = 7; public static int Ext(this int x) => x + 1; public class Inner { } }
    namespace Use
    {
        using static @_;
        static class X
        {
            public static void Run()
            {
                TypeOut.Show("[MIGRATED] 11M-1 `using static @_;` then `Foo`, `new Inner()`", $"{Foo}, {new Inner().GetType().Name}");
                TypeOut.Show("[MIGRATED] 11M-2 extension method in static class `@_`: `1.Ext()`", 1.Ext());
            }
        }
    }
    static class X { public static void Run() => Use.X.Run(); }
}

// 11N: attribute `_`
namespace K14
{
    class @_ : Attribute { }
    [@_] class A1 { }
    [@_] class A2 { }
    [@_()] class A3 { }

    static class X
    {
        public static void Run()
        {
            TypeOut.Show("[MIGRATED] 11N-1 attribute: `[@_]` binds to", string.Join(",", typeof(A1).GetCustomAttributes(false).Select(a => a.GetType().Name)));
            TypeOut.Show("[MIGRATED] 11N-2 attribute: `[@_]` binds to", string.Join(",", typeof(A2).GetCustomAttributes(false).Select(a => a.GetType().Name)));
        }
    }
}

namespace K15
{
    class _Attribute : Attribute { }
    [_Attribute] class A1 { }
    [_Attribute] class A2 { }

    static class X
    {
        public static void Run()
        {
            TypeOut.Show("[MIGRATED] 11O-1 attribute class `_Attribute`: `[_Attribute]` binds to", string.Join(",", typeof(A1).GetCustomAttributes(false).Select(a => a.GetType().Name)));
            TypeOut.Show("[MIGRATED] 11O-2 attribute class `_Attribute`: `[_Attribute]` binds to", string.Join(",", typeof(A2).GetCustomAttributes(false).Select(a => a.GetType().Name)));
        }
    }
}


// =======================================================================================
// Summary of the proposal's effect on each Phase 1 case family
//
//  1  Declarations (locals, foreach, for, using, catch)  : declaration becomes a DISCARD declaration; any READ of `_` => error.
//     Method/local-function/ctor/record parameters, fields, types, enum members, labels, query variables, tuple names, named
//     arguments: error at the declaration; `@_` is the fix.
//  2  Lambdas                                          : every `_` parameter is a discard, whatever the count. 2a-2e become errors where the
//     body reads `_`; `_ => 0` just works as a discard.
//  3  out _                                            : always a discard.  `out @_` for the variable.
//  4  `_ = e`, `(_, x) = e`                            : always a discard assignment.  `@_ = e`, `(@_, x) = e` for the variable.
//  5  Patterns                                         : discard arms/subpatterns/designations unchanged; bare `_` as the whole `is`/`case`
//     pattern stays an error; `case _:` with a constant/type `_` is no longer a constant/type pattern (use `@_`).
//  6  nameof, interpolation, `t._`                     : errors for `_`; `@_` fixes them.
//  7  Type named `_`                                   : declared and used as `@_` everywhere (`_ x` never parsed as a declaration pattern).
//  8  Constant named `_`                               : declared `@_`; `case @_:` is the constant pattern; `_` arms/subpatterns are discards.
//  9  Member named `_`                                 : declared and accessed as `@_`; bare `_ = e`/`out _`/`(_, x)` become discards.
// 10  `@_`                                             : the identifier spelling of `_`; no longer equivalent to bare `_`.
// 11  Type named `_` (any kind)                        : Option Y (proposal as written): every declaration and reference becomes an error; `@_`
//     everywhere is the fix (section 11 above runs the migrated code; outputs match the `_` version in existing-tests.cs).
//     `[_]` that bound to `class _Attribute` also breaks: write `[_Attribute]` (`[@_]` does not probe the `Attribute` suffix).
//     Option X (type `_` stays legal, never a variable): all of section 11 compiles unchanged; `out _`, `_ = e`, `(_, x) = e`
//     stop failing with CS0118 and become discards even when a type `_` is in scope. See the proposal's "Type `_`" section.
// =======================================================================================
```
