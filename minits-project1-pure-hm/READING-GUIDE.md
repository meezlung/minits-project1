# How to read this code

This guide follows one tiny MiniTS program through the whole interpreter. Read it once top to bottom, then keep it open next to the code. Functions are named rather than given line numbers, so use Ctrl+F (or F12 on a name) to jump to them.

The program:

```
let x = 1 + 2;
```

`run` (in `minits.kl`) does three things in order, and each one lives in its own file:

| Stage | File | Input | Output for our program |
|---|---|---|---|
| 1. Parse | `parser.kl` | the text | `Program [SLet "x" None (EBinary OpAdd (ENum 1) (ENum 2))]` |
| 2. Check | `checker.kl` | that AST | OK, and it knows `x : number` |
| 3. Run | `eval.kl` | that AST | `x` lives in box 0, and box 0 holds `3` |

`getGlobal execution "x"` then looks `x` up and gives `Some (VNum 3)`.

---

## 0. Reading KLang: ten things you will see everywhere

| You see | It means |
|---|---|
| `let f x y = body;` | a function `f` with parameters `x`, `y`. `letrec` = it may call itself. |
| `fn x => body` | an anonymous function (a lambda) |
| `f x y` | call `f` with `x` and `y` (no commas, no parentheses) |
| `f (g x)` | parentheses only group: call `g x` first, pass the result to `f` |
| `match v { A a => ...; B => ...; }` | look at what kind of value `v` is and take the matching branch |
| `Ok x` / `Err e` | success with `x` / failure with `e` (our `Result` type, `util.kl`) |
| `Some x` / `None` | there is an `x` / there is nothing (built-in `Option`) |
| `Pair a b` | two things together (our stand-in for a tuple, `util.kl`) |
| `x :: rest`, `[]` | a list whose first item is `x`; the empty list |
| `let y = e in body` | name `e` as `y` inside `body` only |

The pattern you will see most is the **error staircase**:

```
match checkExpr l env {
  Err err => Err err;     # it failed: pass the error up unchanged
  Ok lt =>                # it worked and gave lt: carry on with the next step
    ...
}
```

Read it as "check `l`; if that fails, stop; otherwise call its type `lt` and continue". Nothing more.

---

## 1. The parser: text to AST

### What a parser is

A `Parser a` is a function: give it the input, and it answers either

- `POk value rest err`: "I read a `value`, and this is the text I did not use" (`rest`), or
- `PFail err`: "I could not read one here".

The input is the whole source plus a cursor, so "the text I did not use" just means "the cursor moved". (`err` in `POk` is only kept to give better error messages; ignore it while reading.)

The comments in `parser.kl` write examples like this:

```
charP "a"  on "abc"  =>  ok "a", rest "bc"
```

which means: run `charP "a"` on the text `abc`, and it reads `a` and leaves `bc`.

### The building blocks map one-to-one onto the grammar

The spec writes grammar like `LetStmt ::= "let" Identifier TypeAnn? "=" Expr ";"`. Each grammar symbol has a combinator:

| Grammar | Combinator | Example (from the comments, all tested) |
|---|---|---|
| `p q` (one after the other) | a `do (bindP)` block, or `thenP` / `bindP` | ``charP "(" `thenP` charP "x"`` on `(x)` → ok `"x"`, rest `)` |
| `p \| q` (either) | `orP p q`, `choiceP [...]` | `orP (textP "let") (textP "lex")` on `lex` → ok `"lex"` |
| `p*` (zero or more) | `manyP p` | `manyP (charP "a")` on `aab` → ok `["a", "a"]`, rest `b` |
| `p+` (one or more) | `someP p` | `someP (charP "a")` on `b` → fail |
| `p?` (optional) | `optionP p` | `optionP (charP "a")` on `b` → ok `None`, rest `b` |
| `p ("," p)*` | `sepBy1P p sep` / `sepByP` | `sepBy1P digit (charP ",")` on `1,2,3)` → ok `["1","2","3"]`, rest `)` |
| left-associative `a - b - c` | `chainl1 p op` | `1 - 2 - 3` → `(- (- 1 2) 3)` |
| exact text / one character | `textP "==="`, `charP "a"`, `satisfyP` | `textP "==="` on `=== 1` → ok, rest ` 1` |
| end of the text | `eofP` | |

### Why the grammar never mentions spaces: `lexeme`

`lexeme p` means "read `p`, then skip any spaces and `//` comments after it". Every *token* parser is wrapped in it:

| Token parser | Reads | Then |
|---|---|---|
| `symbolP ";"` | punctuation `( ) { } : ; , .` | skips spaces |
| `operatorP "="` | an operator, but only the whole operator (`=` refuses to be the start of `===` or `=>`) | skips spaces |
| `keywordP "let"` | a keyword, but not the start of a longer word (`let` does not match `letter`) | skips spaces |
| `identifierP` | a name that is not a reserved word | skips spaces |
| `numberP`, `stringP` | literals | skip spaces |

So each token cleans up the spaces behind itself, and the grammar functions only talk about tokens. The spaces before the very first token are skipped once, by `programP`.

### Our program, one line of `letStmtP` at a time

```
let letStmtP =
  do (bindP) {
    keywordP "let";                                 # 1
    let! name = identifierP;                        # 2
    let! ann = optionP (symbolP ":" `thenP` typeP); # 3
    operatorP "=";                                  # 4
    let! init = exprP;                              # 5
    symbolP ";";                                    # 6
    pureP (SLet name ann init)                      # 7
  };
```

| Line | Reads | Result | Text left afterwards |
|---|---|---|---|
| start | | | `let x = 1 + 2;` |
| 1 `keywordP "let"` | `let` and the space | (ignored) | `x = 1 + 2;` |
| 2 `identifierP` | `x` and the space | `name = "x"` | `= 1 + 2;` |
| 3 `optionP (...)` | nothing (there is no `:`) | `ann = None` | `= 1 + 2;` |
| 4 `operatorP "="` | `=` and the space | (ignored) | `1 + 2;` |
| 5 `exprP` | `1 + 2` | `init = EBinary OpAdd (ENum 1) (ENum 2)` | `;` |
| 6 `symbolP ";"` | `;` | (ignored) | (nothing) |
| 7 `pureP` | nothing | `SLet "x" None (EBinary ...)` | |

If any line fails, the whole `letStmtP` fails, and `statementP` (a `choiceP`) tries the next kind of statement from the same starting point.

### Expressions: the precedence ladder

`exprP` reads a whole expression. Each level calls the level below it, so the bottom levels bind tightest:

```
assignmentP      x = e                (loosest, right-associative)
logicalOrP       ||
logicalAndP      &&
equalityP        === !==
comparisonP      < <= > >=
additionP        + -
multiplicationP  * /
unaryP           - !
postfixP         f(x)  e.f
primaryP         numbers, strings, names, (e), arrows, records   (tightest)
```

For `1 + 2`, `additionP` reads `1` from the level below, sees `+`, reads `2`, and builds `EBinary OpAdd (ENum 1) (ENum 2)`. For `1 + 2 * 3`, the `*` is handled one level lower, so it groups first: `(+ 1 (* 2 3))`.

---

## 2. The checker: is the AST well-typed?

Each case of `checkExpr` and `checkStmt` is one typing rule from the spec, labelled with its name. The type environment `TypeEnv` is a stack of scopes (innermost first), each a list of `name: type`.

Our statement `SLet "x" None (EBinary OpAdd (ENum 1) (ENum 2))`, starting from the empty environment `TypeEnv [[]]`:

1. **T-Let** (`checkStmt`, case `SLet`) first checks the initializer.
2. **T-Arith** (`checkExpr`, case `EBinary`): both operands are `ENum`, so both are `TNumber` (**T-Number**). `binaryResultType OpAdd TNumber TNumber` gives `Ok TNumber`.
3. Back in T-Let: `x` is not already in the innermost scope (`declaredLocally`), and there is no annotation to compare against.
4. Result: `bindLocal` gives `TypeEnv [[x: number]]`.

How to match a rule to the code, using T-Let:

| The spec says | The code does |
|---|---|
| Γ ⊢ e : τ | `match checkExpr init env { ... Ok actual => ...` |
| x ∉ local(Γ) | `if declaredLocally env x then Err (DuplicateDeclaration x) else ...` |
| result Γ[x ↦ τ] | `Ok (bindLocal env x actual)` |

If anything fails, the case returns `Err (SomeTypeError ...)` naming the failed requirement, and the staircase passes it straight up to `run`, which wraps it as `TypeFailure`.

---

## 3. The evaluator: running it

Two tables:

- **Env**: name → box number. "Which box does `x` mean here?"
- **Store**: box number → value. "What is in each box right now?"

Our statement, starting from `Env [[]]` and `Store 0 []` (no boxes yet, the next new box is 0):

1. **S-Let** (`execStmt`, case `SLet`): check that `x` is not already declared in this scope, then evaluate the initializer.
2. **E-Binary** (`evalExpr`, case `EBinary`): evaluate `ENum 1` → `VNum 1`, then `ENum 2` → `VNum 2`, then `delta2 OpAdd` → `VNum 3`. The store comes back unchanged.
3. `allocate` makes box 0 holding `VNum 3`: `Store 1 [0 ↦ VNum 3]`.
4. `x` is bound to box 0: `Env [[x ↦ 0]]`.

Why two tables? Assignment changes only the box, not the name. In `let counter = 0; let next = (): number => (counter = counter + 1);`, the function remembers that `counter` means box 0. Every call changes what is *in* box 0, and later reads see the new value.

Every evaluation step returns `Ok (Pair value newStore)`, and the next step uses `newStore`. That is how effects (assignments) flow left to right.

---

## 4. Where do I look when...

| I need to... | Go to |
|---|---|
| add a keyword | `reservedWords` (parser.kl), then a `keywordP "..."` in a new `...StmtP` |
| add a statement | `type Stmt` (syntax.kl) → a new `...StmtP` + `statementP` → a `checkStmt` case → an `execStmt` case |
| add a binary operator | `type BinOp` + `binOpText` → the right level of the ladder → `binaryResultType` → `delta2` |
| add an expression form | `type Expr` + `showExpr` → `primaryP` or a new ladder level → a `checkExpr` case → an `evalExpr` case |
| add a type error | `type TypeError` + `showTypeError` (checker.kl) + `typeErrorKind` (tests.kl) |
| see what the parser built | `parsesTo` / `exprParsesTo` in tests.kl print the AST, e.g. `(+ 1 (* 2 3))` |

Step-by-step recipes with exact line numbers are in `HOPE-TWISTS.md`.

## 5. A reading order that works

1. `syntax.kl`: just the `type` declarations. This is the vocabulary everything else uses.
2. `parser.kl`, Parts 1–2: read the comment above each building block and its examples.
3. `parser.kl`, Part 8 (statements), then Part 7 (expressions): these read like the grammar.
4. `checker.kl`: `binaryResultType`, then `checkExpr` cases one rule at a time.
5. `eval.kl`: `allocate` / `storeGet` / `storeSet`, then `evalExpr` and `execStmt`.
6. `minits.kl`: `run` ties the three stages together.

Skip on a first read: `manyLoop`, `quietP`, `furthest`, `stmtsLoop` and the furthest-error details. They exist for good error messages and termination, not for understanding the grammar.
