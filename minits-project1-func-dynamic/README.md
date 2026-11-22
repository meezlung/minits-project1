# MiniTS interpreter in KLang (`func-dynamic`)

An interpreter for MiniTS, the small statically typed imperative language from CS 150 Project 1.
It parses source text directly, type-checks the whole program, and then runs it.

## Running the tests

1. Select the **`func-dynamic`** dialect (also recorded in `dialect.txt`).
2. Run **`tests.kl`**. It includes every other file through `minits.kl`.

   **Note:** the program is too large for the KLang browser bridge (`klangb`, or **KLang: Run Current File**). The compiled program is over the bridge's 1 MiB message limit, so the run ends with `Bridge messages must not exceed 1048576 UTF-8 bytes` and prints nothing. Instead, run it with the KLang compiler directly, which is kept in this repo: `python -I ../tools/klang/klang-browser.py --run -d func-dynamic tests.kl` (about 40 seconds; see `../tools/klang/README.md`). See `HOPE-TWISTS.md`, section 0.

The last line of `tests.kl` calls `report allTests 0 0;`. That prints one line per test, then a summary:

```
PASS  1c factorial loop: n = 1, result = 120
PASS  1c counter closure: counter = 2, a = 1, b = 2
...
256 passed, 0 failed, 256 total
```

Any failing test prints as `FAIL  <label>`, and the program then ends with `exit 1`.

## Files

Each file includes the one before it, so nothing is defined twice:

| File | Contents |
|---|---|
| `util.kl` | `Result a e` (success first), `Pair`, list helpers, `floatToText` (`Option` is built into KLang) |
| `syntax.kl` | AST: `TypeExpr`, `UnOp`, `BinOp`, `Expr`, `Stmt`, `Program`; printers |
| `parser.kl` | Parser type, combinators, character and text parsers, the grammar, `parse` |
| `checker.kl` | `Type`, `TypeError`, `TypeEnv`, `checkExpr`, `checkStmt`, `checkProgram` |
| `eval.kl` | `Value`, `Env`, `Store`, `Execution`, `RuntimeError`, `evalExpr`, `execStmt`, `execProgram` |
| `minits.kl` | `Error`, `run`, `getGlobal` |
| `tests.kl` | Test suite and `report` |

## Public interface

```
parse        : String -> Result Program ParseError
checkExpr    : Expr -> TypeEnv -> Result Type TypeError
checkStmt    : Stmt -> TypeEnv -> Result TypeEnv TypeError
checkProgram : Program -> Result Unit TypeError
evalExpr     : Expr -> Env -> Store -> Result (Pair Value Store) RuntimeError
execStmt     : Stmt -> Env -> Store -> Result (Pair Env Store) RuntimeError
execProgram  : Program -> Result Execution RuntimeError
run          : String -> Result Execution Error
getGlobal    : Execution -> String -> Option Value
```

`Pair a b` stands in for the tuples in the spec's signatures. `Error` is `ParseFailure ParseError | TypeFailure TypeError | RuntimeFailure RuntimeError`.

Example:

```
match run "let n = 5; let r = 1; while (n > 1) { r = r * n; n = n - 1; }" {
  Ok ex => getGlobal ex "r";      # Some (VNum 120.0)
  Err e => None;
};
```

## Design notes

### Parser (`parser.kl`)
- There is no tokenizer. The parser state is `Input source length offset line column`, and `Parser a` wraps `Input -> Reply a`. Backtracking reruns the other alternative on the old `Input`, which restores both the text position and the line/column.
- `orP` is ordered choice with full backtracking. This lets arrow functions and parenthesized expressions share the `(` prefix: `arrowP` is tried first, and if it fails, `parenP` starts from the same input. The assignment alternative (`Identifier "=" ...`) backtracks the same way.
- `manyP` stops if its parser succeeds without consuming input, so repetition can't loop forever.
- Binary levels use `chainl1`, a left fold, so they are left-associative. Assignment is right-associative because its right-hand side is a full `Expr`. Postfix suffixes are folded in source order.
- Whitespace and `//` comments are skipped after every token (`lexeme`) and once at the start of the program. LF, CR, and CRLF line endings are recognized, including in error positions.
- Keywords must be followed by an identifier boundary, so `letter` is an identifier. Operators are matched by their longest spelling: `=` won't match the start of `===` or `=>`, and `<` won't match the start of `<=`.
- Parse errors carry `line:column`. Each reply tracks the *furthest* failure seen, so an error deep inside an abandoned alternative is still reported where it happened.
- Two rules from the spec text, beyond the bare grammar:
  - A leading `{` at statement position always starts a block, so `{ x: 1 };` is rejected and `({ x: 1 });` is accepted.
  - An arrow body that is a record literal must be parenthesized.

### Type checker (`checker.kl`)
- `TypeEnv` is a persistent stack of scopes, innermost first. `push`, `local` and `Γ[x ↦ τ]` correspond to `pushScope`, `declaredLocally` and `bindLocal`.
- Annotations are written as `TypeExpr` (with function-type parameter names kept) and turned into `Type` by `resolveType`. That step rejects duplicate fields and duplicate function-type parameters, then drops the parameter names.
- `typeEq` compares function types in parameter order with exact return types, and record types by field set regardless of order.
- Each `TypeError` constructor names the requirement that failed, for example `ArityMismatch expected actual` or `MissingField name type`.

### Evaluator (`eval.kl`)
- `Env` maps names to integer locations, and `Store` maps locations to values. `Store next cells` hands out locations from a counter that only increases, so a location is never reused.
- Closures capture the environment, not the store. Calls allocate fresh parameter locations after the arguments are evaluated. Blocks drop their bindings but keep the store.
- `&&` and `||` short-circuit. Everything else evaluates left to right and threads the store.
- Defensive checks return `InvalidRuntimeOperation`, in the order the spec requires: callee shape and arity before the arguments, assignment target before the right-hand side, duplicate declaration before the initializer.
- Assignment also checks that its target's store cell exists before evaluating the right-hand side. This guards manually supplied inconsistent environments; normal program execution allocates every binding.

## Assumptions

- **`Option` comes from KLang itself; `Result` and `Pair` are defined in `util.kl`.** KLang v12 has a built-in `Option a = None | Some a`, and declaring it again is a "duplicate type declaration" compile error. `Result` and `Pair` are not built in.
- **No mutually recursive type declarations** (kept from the typed versions). Named fields and parameters are stored as `Pair name thing`, so every type only refers to itself or to earlier types.
- **Recursion uses plain `let`, as in the func-dynamic labs.** This includes local helpers like `let go ... in`. Mutual recursion is avoided by passing the recursive parser as an argument, as in Lecture 11.
- **The unit value is written `[]`,** matching how func-dynamic prints `Unit`.
- **Number literals use KLang's `string_to_float` once after validating their text and finite range.** This preserves Float rounding and subnormal values. An exact overflow bound is checked first because the native conversion raises an error on non-finite input.
- **MiniTS numbers are KLang `Float`s, so comparisons use the Float operators `<. <=. >. >=.`** (in `eval.kl`'s `delta2` and in `util.kl`'s `floatToText`). KLang's plain `< <= > >=` are Int-only.
