# First-Class `calc()`: Draft 1

*([Issue](https://github.com/sass/sass/issues/2186))*

## Table of Contents

* [Background](#background)
* [Summary](#summary)
  * [Design Decisions](#design-decisions)
    * ["Contagious" Calcs](#contagious-calcs)
    * [Interpolation in `calc()`](#interpolation-in-calc)
    * [Complex Units](#complex-units)
    * [Vendor Prefixed `calc()`](#vendor-prefixed-calc)
* [Syntax](#syntax)
  * [`SpecialFunctionExpression`](#specialfunctionexpression)
  * [`CalcExpression`](#calcexpression)
  * [`CalcValue`](#calcvalue)

## Background

> This section is non-normative.

CSS's [`calc()`] syntax for mathematical expressions has existed for a long
time, and it's always represented a high-friction point in its interactions with
Sass. Sass currently treats `calc()` expressions as fully opaque, allowing
almost any sequence of tokens within the parentheses and evaluating it to an
unquoted string. Interpolation is required to use Sass variables in `calc()`
expressions, and once an expression is created it can't be inspected or
manipulated in any way other than using Sass's string functions.

[`calc()`]: https://drafts.csswg.org/css-values-3/#calc-notation

As `calc()` and related mathematical expression functions become more widely
used in CSS, this friction is becoming more and more annoying. In addition, the
move towards using [`/` as a separator] makes it desirable to use `calc()`
syntax as a way to write expressions using mathematical syntax that can be
resolved at compile-time.

[`/` as a separator]: ../accepted/slash-separator.md

## Summary

> This section is non-normative.

This proposal changes `calc()` (and other supported mathematical functions) from
being parsed as unquoted strings to being parsed in-depth, and sometimes
(although not always) producing a new data type known as a "calc". This data
type represents mathematical expressions that can't be resolved at compile-time,
such as `calc(10% + 5px)`, and allows those expressions to be combined
gracefully within further mathematical functions.

To be more specific: a `calc()` expression will be parsed according to the [CSS
syntax], with additional support for Sass variables, functions, and (for
backwards compatibility) interpolation. Sass will perform as much math as is
possible at compile-time, and if the result is a single number it will return
that number. Otherwise, it will return a calc that represents the (simplified)
expression that can be resolved in the browser.

[CSS syntax]: https://drafts.csswg.org/css-values-3/#calc-syntax

For example:

* `calc(1px + 10px)` will return the number `11px`.

* Similarly, if `$length` is `10px`, `calc(1px + $length)` will return `11px`.

* However, `calc(1px + 10%)` will return the calc `calc(1px + 10%)`.

* If `$length` is `calc(1px + 10%)`, `calc(1px + $length)` will return
  `calc(2px + $length)`.

* Sass functions can be used directly in `calc()`, so `calc(1% +
  math.round(15.3px))` returns `calc(1% + 15px)`.

Note that calcs cannot generally be used in place of numbers. For example,
`1px + calc(1px + 10%)` will produce an error, as will `math.round(calc(1px +
10%))`.

For backwards compatibility, `calc()` expressions that contain interpolation
will continue to be parsed using the old highly-permissive syntax, although this
behavior will eventually be deprecated and removed. These expressions will still
return calc values, but they'll never be simplified or resolve to plain numbers.

### Design Decisions

#### "Contagious" Calcs

#### Interpolation in `calc()`

#### Complex Units

#### Vendor Prefixed `calc()`

## Syntax

### `SpecialFunctionExpression`

This proposal replaces the definition of [`SpecialFunctionName`] with the
following:

[`SpecialFunctionName`]: ../spec/syntax.md#specialfunctionexpression

<x><pre>
**SpecialFunctionName**¹      ::= VendorPrefix? ('element(' | 'expression(')
&#32;                           | VendorPrefix `calc('
</pre></x>

1: The string `calc(` is matched case-insensitively.

### `CalcExpression`

This proposal defines a new production `CalcExpression`. This expression should
be parsed in a SassScript context when an expression is expected and the input
stream starts with an identifier with value `calc` (ignoring case) followed
immediately by `(`.

The grammar for this production is:

<x><pre>
**CalcExpression** ::= `calc(`¹ CalcArgument ')'
**CalcArgument**²  ::= InterpolatedDeclarationValue | CalcSum
**CalcSum**     ::= CalcProduct (('+' | '-') CalcProduct)?
**CalcProduct** ::= CalcValue (('*' | '/') CalcValue)?
**CalcValue**   ::= '(' CalcSum ')
&#32;             | CalcExpression
&#32;             | CssMinMax
&#32;             | FunctionExpression³
&#32;             | Number
&#32;             | Variable
&#32;             | Interpolation
</pre></x>

1: The string `calc(` is matched case-insensitively.

2: A `CalcArgument` is only parsed as an `InterpolatedDeclarationValue` if it
includes interpolation, unless that interpolation is within a region bounded by
a [`<function-token>`] on the left and an unbalanced ")" on the right.

3: This `FunctionExpression` cannot begin with `min(` or `max(`,
case-insensitively.

[`<function-token>`]: https://drafts.csswg.org/css-syntax-3/#ref-for-typedef-function-token%E2%91%A0

> The `CalcArgument` production provides backwards-compatibility with the
> historical use of interpolation to inject SassScript values into `calc()`
> expressions. Because interpolation could inject any part of a `calc()`
> expression regardless of syntax, for full compatibility it's necessary to
> parse it very expansively.
>
> Note that the interpolation in the definition of `CalcValue` is only reachable
> from a `CssMinMax` production, *not* from `CalcExpression`. This is
> intentional, for backwards-compatibility with the existing `CssMinMax` syntax.x

### `CssMinMax`

This proposal replaces the reference to `CalcValue` in the definition of
[`CssMinMax`] with `CalcSum`.

[`CssMinMax`]: ../spec/syntax.md#MinMaxExpression

## Types

This proposal introduces a new value type known as a "calc", with the following
structure:

```ts
interface Calc {
  name: string;
  arguments: CalcValue[];
}

type CalcValue =
  | Number
  | UnquotedString
  | CalcOperation
  | Calc;

interface CalcOperation {
  operator: '+' | '-' | '*' | '/';
  left: CalcValue;
  right: CalcValue;
}
```

Unless otherwise specified, when this specification creates a calc, its name is
"calc".

### Operations

A calc follows the default behavior of all SassScript operations, except that it
throws an error if used as an operand of a unary or binary `+` or `-` operation.

> This helps ensure that if a user expects a number and receives a calc instead,
> it will throw an error quickly rather than propagating as an unquoted string.

### Serialization

To serialize a calc, emit its name followed by "(", then each of its arguments
separated by ",", then ")".

Each argument is serialized in the standard manner of its type, except for a
`CalcOperation` which is serialized as follows:

* Let `left` and `right` be the result of serializing the left and right values,
  respectively.

* If the operator is `*` or `/` and the left value is a `CalcOperation` with
  operator `+` or `-`, emit "(" followed by `left` followed by ")". Otherwise,
  emit `left`.

* Emit " ", then the operator, then " ".

* If the operator is `*` or `/` and the right value is a `CalcOperation` with
  operator `+` or `-`, emit "(" followed by `right` followed by ")". Otherwise,
  emit `right`.

## Semantics

### `CalcExpression`

To evaluate a `CalcExpression`:

* If the expression's `CalcArgument` is an `InterpolatedDeclarationValue`,
  evaluate it and return a calc with the resulting unquoted string as its only
  argument.

* Otherwise, return a calc whose only argument is the result of [evaluating the
  `CalcArgument`'s `CalcValue`].

[evaluating the `CalcArgument`'s `CalcValue`]: #calcvalue

### `MinMaxExpression`

### `CalcSum`

To evaluate a `CalcSum` production `sum` into a `CalcValue` object:

* Left `left` be the result of evaluating the first `CalcProduct`.

* For each remaining "+" or "-" token `operator` and `CalcProduct` `product`:

  * Let `right` be the result of evaluating `product`.

  * Set `left` to a `CalcOperation` with `operator`, `left`, and `right` as its
    values.

* Return `left`.

### `CalcProduct`

To evaluate a `CalcProduct` production `product` into a `CalcValue` object:

* Left `left` be the result of evaluating the first `CalcValue`.

* For each remaining "*" or "/" token `operator` and `CalcValue` `value`:

  * Let `right` be the result of evaluating `value`.

  * Set `left` to a `CalcOperation` with `operator`, `left`, and `right` as its
    values.

* Return `left`.

### `CalcValue`

To evaluate a `CalcValue` production `value` into a `CalcValue` object:

* If `value` is a `CalcSum`, `CssMinMax`, or `Number`, return the result of
  evaluating it. These are guaranteed to evaluate to `CalcValue`s.

* If `value` is an `Interpolation`, evaluate it and return it as an unquoted
  string.

* If `value` is a `FunctionExpression` or `Variable`, evaluate it. If the result
  is a number or an unquoted string, return it. Otherwise, throw an error.

> Allowing variables to return unquoted strings here supports referential
> transparency, so that `$var: fn(); calc($var)` works the same as `calc(fn())`.

## Functions

### `meta.type-of()`

Add the following clause to the [`meta.type-of()`] function and the top-level
`type-of()` function:

[`meta.type-of()`]: ../spec/built_in_modules/meta.md#type-of

* If `$value` is a calc, return `"calc"`.

### `meta.calc-name()`

### `meta.calc-args()`

TODO: How do we deal with this?

```scss
$px-per-pc: 10px;
$calc: clac($px-per-pc * 1% / 1px);
```

