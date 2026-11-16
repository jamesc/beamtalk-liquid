# liquid

A runtime, parse-then-render [Liquid](https://shopify.github.io/liquid/)-subset
template engine for [Beamtalk](https://beamtalk.dev), modelled on Elixir's
[Solid](https://github.com/edgurgel/solid).

Templates are data, not code: only the tags and filters listed below can run.
Rendered output is never re-scanned, so a variable whose value contains
`{{ ... }}` or `{% ... %}` is emitted literally.

## Installation

```toml
[dependencies]
liquid = { git = "https://github.com/jamesc/beamtalk-liquid", tag = "v0.1.0" }
```

## Usage

```beamtalk
(Liquid parse: "Hello {{ issue.title | upcase }}") andThen: [:template |
  template render: #{"issue" => #{"title" => "fix it"}}]
// => Result ok: "Hello FIX IT"
```

- `Liquid parse: source :: String` returns `Result(Template, LiquidError)`.
  Parse once, render many times.
- `Template render: vars :: Dictionary` returns `Result(String, LiquidError)`.
- `Template render: vars options: #{#strictVariables => true}` records
  undefined variables as errors (see [Strict variables](#strict-variables)).
  Unknown option keys and non-Boolean values are errors.

Variable maps use **string keys** (`#{"issue" => ...}`), as produced by
`asJson`. Symbol keys are not looked up.

In Beamtalk source, write template braces as `\{\{ x }}` and `\{% if x %}`:
`{` starts string interpolation in a Beamtalk string literal.

## Supported subset

**Output:** `{{ expr }}` where `expr` is a literal (`"s"`, `'s'`, `1`, `-2`,
`1.5`, `true`, `false`, `nil`, `null`) or a path (`a.b`, `a[0]`, `a["k"]`,
`a[var]`), followed by any number of filters.

**Tags:** `if` / `elsif` / `else` / `endif`, `unless` / `elsif` / `else` /
`endunless`, `for` / `else` / `endfor`, `comment` / `endcomment`,
`raw` / `endraw`. Blocks nest.

**Whitespace control:** `{%-` and `{{-` strip all whitespace (including
newlines) before the tag; `-%}` and `-}}` strip all whitespace after it.

**Conditions:** operators `==  !=  <>  <  >  <=  >=  contains  and  or`.
`<>` is an alias of `!=`.

**`forloop`:** `index`, `index0`, `rindex`, `rindex0`, `first`, `last`,
`length`, `parentloop`, `name`.

**Filters** (names and semantics as in Solid's `standard_filter.ex`):

| Filter | Behaviour |
|---|---|
| `default: v` | `v` (default `""`) when the input is `nil`, `false`, `""`, `[]` or `{}`; `0` is kept |
| `upcase`, `downcase` | Unicode-aware case change of the stringified input |
| `strip` | trims leading and trailing whitespace |
| `truncate: n, ellipsis` | `n` defaults to 50, ellipsis to `"..."`; the ellipsis counts towards `n` |
| `join: glue` | glue defaults to `" "`; a non-list is a one-element list; nested lists flatten |
| `size` | length of a string (characters), list or map; `0` for anything else |
| `first`, `last` | first / last element of a list or character of a string; `nil` otherwise |
| `escape` | HTML-escapes `& < > " '` |

Path helpers: `.size` works on lists, strings and maps (a real `size` key wins);
`.first` / `.last` work on lists; list indexes are 0-based and `-1` counts from
the end.

### Semantics worth knowing

- **Truthiness:** only `nil` and `false` are falsy. `0`, `""`, `[]` and `{}` are
  truthy.
- **`and` / `or` have no precedence and are right-associative:**
  `a and b or c` means `a and (b or c)`, and `a or b and c` means
  `a or (b and c)`. Use nested `if` tags to get other groupings. Both sides are
  always evaluated (no short-circuit), so strict-variable errors on either side
  are reported.
- **`contains`:** a string on the left does a substring check (a non-string
  right side is converted to text first); a list on the left tests membership;
  `nil` on either side is `false`; any other left type is `false`.
- **Ordering (`< > <= >=`):** `nil` on either side is `false`; a string against
  a number is a render error (`comparison of String with 3 failed`); two numbers
  or two strings compare normally; any other pairing is `false`.
- **Maps compare `false` for every operator** (including `==` and `!=`).
- **`==` is loose between numbers:** `1 == 1.0` is true.
- **`unless`:** the first branch runs when its condition is falsy; `elsif` and
  `else` act as fallbacks, as for `if`.
- **Extra `else` / `elsif` after an `else` is ignored**, as in Solid: the extra
  clause is parsed and dropped.
- **`comment`:** the body is skipped by scanning for `endcomment`. Tags inside are
  not parsed and malformed markup inside is not an error.
- **`raw`:** the body is emitted literally up to `endraw`. `{%- endraw` trims the
  body's trailing whitespace and `{% raw -%}` trims its leading whitespace.
- **Strings have no escapes.** A `"` string cannot contain `"`; use `'` for that.
  Numbers are integers or floats; a float needs a digit after the dot.
- **Identifiers** may contain `?` and `-` (`my-var`, `ok?`), as in Solid.
- **Output conversion:** `nil` is empty; lists concatenate their elements; maps
  use their printString; numbers and booleans use their display form.

### `for`

```liquid
{% for item in items %}{{ forloop.index }}. {{ item }}
{% else %}nothing{% endfor %}
```

Choices made where the spec leaves room (listed in the PR for the owner):

- `nil` (or an undefined variable) renders the `else` body.
- A scalar (string, number, boolean) iterates once as a one-element list
  (Liquid behaviour).
- A map iterates `[key, value]` pairs (`pair[0]`, `pair[1]`), in the map's
  key order.
- An empty list or map renders the `else` body.

### Strict variables

With `#{#strictVariables => true}`, a variable path that does not exist is
recorded as an `#undefined_variable` error at the position of the variable. The
variable renders as an empty string, rendering continues, and `render:` returns
an error result carrying every recorded error. The partial text is not
returned.

"Does not exist" means the key is absent (or an index is out of range, or a
parent on the path is `nil`). A key that is present with value `nil`, `false`
or `0` is defined. Only variables that are actually evaluated count, so a typo
inside an untaken `if` branch is not reported. Filters such as
`default` do not rescue an undefined variable in strict mode.

### Errors

`LiquidError` is a tagged error value with:

| Field | Meaning |
|---|---|
| `kind` | `#lex`, `#parse`, `#unknown_tag`, `#unknown_filter`, `#unclosed`, `#undefined_variable`, `#render` |
| `line`, `column` | 1-based; **columns count Unicode codepoints**, not bytes |
| `detail` | the message, e.g. `Expected 'endif'`, `Filter expected`, `Unknown tag 'x'` |

Parse and strict-render failures can have several errors (Solid accumulates
them and continues). **Choice:** a `LiquidError` describes the *first* error by
source position in its own fields and also carries the others: `errors` returns
every error (primary first, each a LiquidError), `count` their number, and
`rest` the remaining ones. `message` renders `kind at line:column: detail`.
Callers that only care about one error can ignore the list.

Within a single tag parsing stops at the first error; the parser then resumes at
the next tag, so independent mistakes in one template are all reported.

## Known divergences from Liquid and Solid

Not supported (deferred): `assign`, `capture`, `case`, `render` / `include`,
`date`, counters, `tablerow`, `cycle`, `echo`, `for` parameters (`limit`,
`offset`, `reversed`), `break` / `continue`, ranges (`(1..5)`), the `blank` and
`empty` literals, `strict_filters`, and every filter not listed above. A tag
such as `{% assign x = 1 %}` is an error (`#lex` for the `=`, or `#unknown_tag`).

Behaviour that differs:

- **Unknown filters are a parse error** (`#unknown_filter`). Solid passes the
  value through unchanged unless `strict_filters` is set.
- **Filter arity is checked at parse time** (`upcase: 1` is an error); Solid
  checks at render time. Named filter arguments (`default: x, allow_false: true`)
  are not supported.
- **No filters inside `if` conditions** (Solid allows them behind an option).
- **Error columns count codepoints**; Solid counts bytes.
- **Strict mode returns an error instead of partial text**; Solid returns the
  partial result alongside the errors.
- **`or` propagates a child error** like `and` does. Solid's `or` branch uses
  `=` where `and` uses `<-` (`condition_expression.ex`), so a child error raises
  a `MatchError` there.
- **`first` / `last` on a map** return `nil`; Solid returns a `[key, value]` pair.
- **`nil` in ordering comparisons is always `false`**, including against
  strings. (Solid only special-cases nil against numbers; against strings it
  falls through to Elixir term ordering.)
- **Map keys are strings only.** The engine never creates atoms from template
  text.
- **No `Decimal` arithmetic**; numbers are Beamtalk integers and floats.
- **`forloop.name`** is `"<var>-<collection path>"` as in Liquid; for a literal
  collection it is `"<var>-literal"`.

## Provenance of the semantics

Condition, `for`, `comment`, `raw` and strict-variable semantics follow Solid
(`lexer.ex`, `parser.ex`, `binary_condition.ex`, `condition_expression.ex`,
`tags/if_tag.ex`). Filter semantics were checked against the function bodies
and doctests in Solid's `lib/solid/standard_filter.ex` (Solid's
`test/solid/standard_filter_test.exs` covers only `apply/4` dispatch, not
individual filters).

The leading side of whitespace control (`{%-` trims before the tag), `forloop`
first/last/length, truthiness, `comment`, `raw` and nested `if` are specified
from the Liquid reference rather than from Solid's tests, which do not cover
them; the tests for them are written from that spec.

## Development

```bash
just build
just test
just ci
```

## License

Apache-2.0
