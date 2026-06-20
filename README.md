<h1 align="center">parser_combinator.mq</h1>

A small parser-combinator toolkit for [mq](https://github.com/harehare/mq), in the spirit of Rust's [nom](https://github.com/rust-bakery/nom). It gives you the building blocks (`pc_char`, `pc_tag`, `pc_regex`, ...) and combinators (`pc_map`, `pc_many0`, `pc_alt`, `pc_sep_by0`, ...) to write small, composable parsers directly in mq, instead of hand-rolling string slicing and index bookkeeping every time.

## Concept

A **parser** is just a function value with the signature `fn(input, pos): result;`. `result` is one of:

```mq
{ok: true,  pos: <new position>, value: <parsed value>, error: None}
{ok: false, pos: <position of failure>, value: None,    error: <message>}
```

Combinators take one or more parsers and return a **new parser**, so they compose exactly like nom:

```mq
let digits = pc_map(pc_take_while1(_pc_is_digit), to_number)
```

Because parsers are ordinary mq function values, recursive grammars are just ordinary recursive `def`s — see [Recursive grammars](#recursive-grammars) below.

## Installation

Copy `parser_combinator.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp parser_combinator.mq ~/.local/mq/config/
```

### HTTP Import (no local installation needed)

If `mq` was built with the `http-import` feature, you can import directly from GitHub without any local setup:

```sh
mq -I raw 'import "github.com/harehare/parser_combinator.mq" | parser_combinator::pc_run(parser_combinator::pc_tag("hello"), .)' input.txt
```

Pin to a specific release with `@vX.Y.Z`:

```sh
mq -I raw 'import "github.com/harehare/parser_combinator.mq@v0.1.0" | ...' input.txt
```

## Usage

```sh
mq -L /path/to/modules -I raw \
  'include "parser_combinator" | pc_run(pc_many1(pc_digit()), .)' input.txt
```

## API

### Result helpers

| Function | Description |
|---|---|
| `pc_ok(pos, value)` | Builds a success result. Use it when writing a custom parser by hand. |
| `pc_err(pos, message)` | Builds a failure result. |
| `pc_pos_to_line_col(input, pos)` | Returns `{line, col}` (1-based) for a position, for error messages. |

### Primitives

Each of these is a constructor — it returns a parser, it is not itself a parser.

| Function | Description |
|---|---|
| `pc_char(c)` | Matches one exact character. |
| `pc_any_char()` | Matches any single character. Fails at end of input. |
| `pc_tag(s)` | Matches a literal string (nom's `tag`). |
| `pc_satisfy(pred)` | Matches a single character for which `pred(ch)` is true. |
| `pc_one_of(chars)` | Matches a single character that appears in `chars`. |
| `pc_none_of(chars)` | Matches a single character that does **not** appear in `chars`. |
| `pc_digit()` | Matches a single ASCII digit. |
| `pc_alpha()` | Matches a single ASCII letter. |
| `pc_alnum()` | Matches a single alphanumeric ASCII character. |
| `pc_space()` | Matches a single whitespace character. |
| `pc_regex(pattern)` | Matches the regex `pattern`, implicitly anchored at the current position. |
| `pc_eof()` | Succeeds only at the end of input. Consumes nothing. |
| `pc_rest()` | Always succeeds, consuming the rest of the input. |
| `pc_take(n)` | Consumes exactly `n` characters. |
| `pc_take_while(pred)` | Consumes characters while `pred(ch)` holds. Always succeeds (may match zero). |
| `pc_take_while1(pred)` | Like `pc_take_while`, but fails on zero matches. |
| `pc_take_until(s)` | Consumes up to (not including) the first occurrence of `s`. |
| `pc_whitespace0()` | Zero or more whitespace characters. |
| `pc_whitespace1()` | One or more whitespace characters. |

### Transforming a parser's value

| Function | Description |
|---|---|
| `pc_map(p, f)` | Transforms `p`'s value with `f` on success. |
| `pc_value(p, v)` | Replaces `p`'s success value with the constant `v`. |
| `pc_recognize(p)` | Replaces `p`'s value with the substring it consumed. |

### Lookahead & optionality

| Function | Description |
|---|---|
| `pc_opt(p)` | Tries `p`; always succeeds with `p`'s value or `None`. |
| `pc_not(p)` | Negative lookahead: succeeds (consuming nothing) only if `p` fails. |
| `pc_peek(p)` | Lookahead: runs `p` but never consumes input. |

### Sequencing

| Function | Description |
|---|---|
| `pc_pair(p1, p2)` | Runs `p1` then `p2`. Value is `[v1, v2]`. |
| `pc_preceded(p1, p2)` | Runs `p1` then `p2`, keeping only `p2`'s value. |
| `pc_terminated(p1, p2)` | Runs `p1` then `p2`, keeping only `p1`'s value. |
| `pc_delimited(open, p, close)` | Runs `open`, `p`, `close`, keeping only `p`'s value. |
| `pc_seq(parsers)` | Runs every parser in the array `parsers` in order. Value is an array. |
| `pc_alt(parsers)` | Ordered choice: tries each parser in the array `parsers`, returns the first success. |

### Repetition

| Function | Description |
|---|---|
| `pc_many0(p)` | Zero or more `p`. Always succeeds. Value is an array. |
| `pc_many1(p)` | One or more `p`. Fails on zero matches. |
| `pc_count(p, n)` | Exactly `n` repetitions of `p`. |
| `pc_sep_by0(p, sep)` | Zero or more `p` separated by `sep`. |
| `pc_sep_by1(p, sep)` | One or more `p` separated by `sep`. |
| `pc_many_till(p, end_p)` | Repeats `p` until `end_p` succeeds. Value is `{items: [...], end: <end_p's value>}`. |

### Running a parser

| Function | Description |
|---|---|
| `pc_parse(p, input)` | Runs `p` from position 0. Returns the raw `{ok, pos, value, error}` result; the input need not be fully consumed. |
| `pc_run(p, input)` | Runs `p` and requires the **entire** input to be consumed. Returns the value directly on success, or raises a descriptive error (with line/column) on failure. |

## Example: parsing comma-separated numbers

```sh
printf '1,2,3' | mq -I raw 'include "parser_combinator" | pc_run(pc_sep_by1(pc_map(pc_take_while1(_pc_is_digit), to_number), pc_char(",")), .)'
# => [1, 2, 3]
```

## Example: an arithmetic expression evaluator

This is the kind of small, ad-hoc parser this module is meant to make easy to write — a four-operator calculator with correct precedence and parentheses, built entirely from the combinators above:

```mq
include "parser_combinator"
|

def ws(p): pc_terminated(p, pc_whitespace0());

def number():
  ws(pc_map(pc_take_while1(_pc_is_digit), to_number))
end

def apply_op(acc, pair):
  let op = pair[0]
  | let rhs = pair[1]
  | if (op == "+"): acc + rhs
    elif (op == "-"): acc - rhs
    elif (op == "*"): acc * rhs
    else: acc / rhs
end

# expr := term (("+" | "-") term)*
def expr(input, pos):
  let p = pc_pair(term, pc_many0(pc_pair(ws(pc_one_of("+-")), term)))
  | let r = p(input, pos)
  | if (!r["ok"]): r else: pc_ok(r["pos"], fold(r["value"][1], r["value"][0], apply_op))
end

# term := factor (("*" | "/") factor)*
def term(input, pos):
  let p = pc_pair(factor, pc_many0(pc_pair(ws(pc_one_of("*/")), factor)))
  | let r = p(input, pos)
  | if (!r["ok"]): r else: pc_ok(r["pos"], fold(r["value"][1], r["value"][0], apply_op))
end

# factor := number | "(" expr ")"
def factor(input, pos):
  let p = pc_alt([number(), pc_delimited(ws(pc_char("(")), expr, ws(pc_char(")")))])
  | p(input, pos)
end

| pc_run(ws(expr), "2 * (3 + 4) - 5")
# => 9
```

`expr`, `term`, and `factor` call each other before all three are fully defined — that's fine, named `def`s in mq resolve by name at call time, so mutually recursive grammars are written exactly like you'd write them on paper.

## Recursive grammars

Combinators like `pc_many0` and `pc_alt` take a parser *value*. To build a parser that refers to itself (lists of lists, nested expressions, etc.), write a named `def` with the parser signature `(input, pos)` and call it by name — recursion and forward references between `def`s work without any extra setup:

```mq
def item(input, pos):
  let p = pc_alt([list_, number_()])
  | p(input, pos)
end

def list_(input, pos):
  let p = pc_delimited(pc_char("["), pc_sep_by0(item, pc_char(",")), pc_char("]"))
  | p(input, pos)
end

def number_():
  pc_map(pc_take_while1(_pc_is_digit), to_number)
end

| pc_run(list_, "[1,[2,3],4]")
# => [1, [2, 3], 4]
```

## A note on syntax

mq does not allow chaining a call directly onto a call's result, e.g. `pc_many0(p)(input, pos)`. Bind the constructed parser to a name first:

```mq
let many_digits = pc_many0(pc_digit())
| many_digits(input, pos)
```

## License

MIT
