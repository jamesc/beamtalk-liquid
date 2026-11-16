# liquid

A runtime, parse-then-render [Liquid](https://shopify.github.io/liquid/)-subset
template engine for [Beamtalk](https://beamtalk.dev), modelled on Elixir's
[Solid](https://github.com/edgurgel/solid).

> **Status:** scaffolding. The engine is being built in BT-4918.

## Installation

```toml
[dependencies]
liquid = "0.1.0"
```

## Usage (planned)

```beamtalk
(Liquid parse: "Hello {{ issue.title | upcase }}") andThen: [:template |
  template render: #{"issue" => #{"title" => "fix it"}}]
// => Result ok: "Hello FIX IT"
```

- `Liquid parse: source :: String` returns `Result(Template, LiquidError)`.
- `Template render: vars :: Dictionary` returns `Result(String, LiquidError)`.
- `Template render: vars options: #{#strictVariables => true}` reports undefined
  variables as errors.

## Supported subset (planned)

- Output: `{{ expr }}` with `|` filters and `name: arg, arg` arguments.
- Tags: `if` / `elsif` / `else`, `unless`, `for` (with `forloop` and `else`),
  `comment`, `raw`.
- Whitespace control: `{%-`, `-%}`, `{{-`, `-}}`.
- Operators: `== != <> < > <= >= contains and or`. `and` / `or` have no
  precedence and are right-associative: `a and b or c` means `a and (b or c)`.
- Truthiness: only `nil` and `false` are falsy.
- Filters: `default`, `upcase`, `downcase`, `strip`, `truncate`, `join`,
  `size`, `first`, `last`, `escape`.

## Known divergences from Liquid

To be filled in as the engine lands. Not supported: `assign`, `capture`,
`case`, `render` / `include`, `date`, counters, `tablerow`, `for` parameters
(`limit`, `offset`, `reversed`), `break` / `continue`, `blank` / `empty`.

## Development

```bash
just build
just test
just ci
```

## License

Apache-2.0
