# HOPE twist playbook (pure-hm)

This file covers how to change this interpreter when a HOPE twist asks for a new feature: which file, which line, and what code to add.

All line numbers are for the **unmodified** files in this folder. Once you insert lines for one twist, every later line in that file shifts down. When you do two twists, find the spot by the function name given next to each line number (Ctrl+F) rather than by the number.

Every recipe below was compiled and run with the real KLang 12.0.0 compiler. Ten twists have complete reference implementations in `../minits-hope-twists/pure-hm/<name>/`, written in **pure-hm** like this folder, so you can copy their code directly. The same ten twists exist for the other dialects in `../minits-hope-twists/<dialect>/<name>/`.

---

## 0. Running and testing

| Where | Command | Works? |
|---|---|---|
| KLang Web IDE (`klang.upd-dcs.work`), dialect **pure-hm** | open `tests.kl`, Run | yes (it compiles in the page) |
| VS Code **KLang: Run Current File** / `klangb --run` | | **no**: the compiled program is over the bridge's 1 MiB message cap, so it prints nothing and exits with `Bridge messages must not exceed 1048576 UTF-8 bytes` |
| The compiler itself, kept in this repo (`../tools/klang/klang-browser.py`, KLang 12.0.0; see `../tools/klang/README.md`) | `python -I ../tools/klang/klang-browser.py --run -d pure-hm tests.kl` | yes, about 40 s |

The output is one `PASS`/`FAIL` line per test, then a summary like `242 passed, 0 failed, 242 total`. Each twist folder has more tests (247 to 252).

**Tip for the exam:** when you add a twist, add 4 to 8 tests for it (section 4 has a template) and rerun. A twist that breaks an old test usually broke the parser, for example by making a word reserved or by changing what an operator character can be followed by.

---

## 1. How the code is laid out

Pipeline: `run src` (`minits.kl:14`) = **parse** → **checkProgram** → **execProgram**. Each stage stops at its first error: `ParseFailure` / `TypeFailure` / `RuntimeFailure`.

| Stage | File | What to look at |
|---|---|---|
| AST | `syntax.kl` | `type BinOp` :31, `type Expr` :37, `type Stmt` :54, printers `binOpText` :75, `showExpr` :135, `showStmt` :169 |
| Parser combinators | `parser.kl` | `bindP` :105, `orP` :131 (backtracks), `choiceP` :143, `manyP` :164, `optionP` :172, `sepByP` :180, `chainl1` :185, `notFollowedByP` :193 |
| Tokens | `parser.kl` | `wsP` :282 (whitespace + comments), `symbolP` :296, `operatorP` :302 (longest-match rules), `reservedWords` :315, `keywordP` :328, `identifierP` :335 |
| Literals | `parser.kl` | `numberRawP` :363, `escapeChar` :385, `stringBody` :395 |
| Types (syntax) | `parser.kl` | `primitiveTypeP` :439, `functionTypeP` :453, `recordTypeP` :466 |
| Expressions, by precedence (loosest at the bottom) | `parser.kl` | `primaryP` :533 → `postfixP` :570 → `unaryP` :576 → `multiplicationP` :590 → `additionP` :594 → `comparisonP` :598 → `equalityP` :603 → `logicalAndP` :607 → `logicalOrP` :611 → `assignmentP` :617 |
| Statements | `parser.kl` | `blockP` :651, `letStmtP` :657, `ifStmtP` :666, `whileStmtP` :681, `exprStmtP` :689, **`statementP` :694** (list of alternatives) |
| Types (semantic) | `checker.kl` | `type Type` :9, **`type TypeError` :18**, `showTypeError` :179, `typeEq` :106, `resolveType` :211 |
| Operator typing | `checker.kl` | **`binaryResultType` :290** (one line per operator) |
| Expression typing | `checker.kl` | `checkExpr` :328 (`EAssign` :376, `EUnary` :387, `EBinary` :398, `EFun` :409, `ECall` :436, `EField` :454, `ERecord` :469) |
| Statement typing | `checker.kl` | `checkStmt` :496 (`SLet` :510, `SBlock` :537, `SIf` :544, `SWhile` :559) |
| Values / store | `eval.kl` | `type Value` :11, `type RuntimeError` :33, `allocate` :94, `storeSet` :112 |
| Operators at runtime | `eval.kl` | `delta1` :165 (unary), **`delta2` :189** (strict binary) |
| Expression eval | `eval.kl` | `evalExpr` :257 (`EAssign` :292, `EBinary` :313 with `&&`/`||` short-circuit, `ECall` :379) |
| Statement eval | `eval.kl` | `execStmt` :438 (`SLet` :453, `SBlock` :473, `SIf` :480, `SWhile` :496), `execProgram` :529 |
| Tests | `tests.kl` | `typeErrorKind` :112, `defensiveTests` :625, `# Running` :653, `allTests` :656 |

### Four ideas the whole design rests on

1. **The parser works on raw text, with no lexer.** `Input src len offset line col` is the cursor. `orP p q` reruns `q` on the *old* input when `p` fails, so backtracking is free. `lexeme p` = run `p`, then skip whitespace/comments. Every token parser (`symbolP`, `operatorP`, `keywordP`, `identifierP`, `numberP`, `stringP`) already skips trailing whitespace.
2. **Precedence is a ladder of functions.** Each level calls the next-tighter level and `chainl1`s its operators. This makes them left-associative. To add an operator at an existing level, add one `binOpP "text" OpX` to that level's alternatives.
3. **Desugaring.** If a new construct can be written with existing ones, build the old AST in the parser and leave the checker and evaluator alone. `else if`, `for`, `do-while`, and compound assignment `+=` all work this way.
4. **Two environments, same shape.** The checker's `TypeEnv` maps name→type and the evaluator's `Env` maps name→location. Both are a stack of scopes, innermost first. `Store` maps location→value, so assignment changes the store and never the environment.

---

## 2. Writing KLang in this folder (pure-hm)

- **A recursive top-level function needs `letrec`.** Plain top-level `let` cannot refer to itself ("unbound name"). A local helper with parameters (`let go x = ... in`) may recurse with plain `let`.
- **The unit value is `unit`.**
- **Effects are IO values.** Use `outputLnIO`, `bindIO`, `pureIO`, and the program is `let main = ...;` (`tests.kl:677`). The interpreter itself (`parse`, `run`, ...) is pure and never prints.
- Code in the reference folders (`../minits-hope-twists/pure-hm/*`) is written in this dialect and can be pasted as-is.

Common to all three dialects (KLang 12.0.0):

- **Float vs Int operators.** MiniTS numbers are `Float`, so use `+. -. *. /.` and `<. <=. >. >=.`. Plain `+ - * / < <= > >=` are **Int-only** (offsets, lengths, `str_ord`). Getting this wrong was the real compiler error in this project: `x < y` in `delta2` and `x < 0.0` in `floatToText`. `==` and `!=` work on any type.
- **`Option` is built in** (`None | Some a`). Do not declare it, or you get a "duplicate type declaration" error. `Result` and `Pair` are ours, in `util.kl`.
- **`&&` / `||` in KLang evaluate both sides.** Never write `ok && recursiveCall ...` expecting a short-circuit. Use `if ... then ... else false`. MiniTS's own `&&`/`||` do short-circuit; that is done explicitly in `eval.kl:313`.
- Useful built-ins: `str_len`, `str_at s i`, `str_slice s from to` (it errors if out of range), `str_concat`, `str_join [..]`, `str_ord`, `str_compare a b` (-1/0/1), `int_to_float`, `float_to_int` (truncates toward 0), `float_to_string`, `int_to_string`.
- Patterns can be nested: `Err (LoopBreak st) => ...`, `Some (VNum n) => ...`, `VBool true => ...`.

---

## 3. Checklist for any twist

Ask: **is it new syntax only (sugar), a new operator, a new expression, or a new statement?**

| Kind | syntax.kl | parser.kl | checker.kl | eval.kl | tests.kl |
|---|---|---|---|---|---|
| Sugar (else-if, for, do-while, `+=`) | – | ✔ build existing AST | – | – | ✔ |
| New binary operator (`%`) | `BinOp` + `binOpText` | one `binOpP` at the right precedence level | one line in `binaryResultType` | one line in `delta2` | ✔ |
| New unary operator | `UnOp` + `unOpText` | `unaryP` :576 | `EUnary` case :387 | `delta1` :165 | ✔ |
| New expression (`?:`) | `Expr` + `showExpr` | new level, or new `primaryP` alternative | new `checkExpr` case (+ maybe a new `TypeError`) | new `evalExpr` case | ✔ |
| New statement (`const`, `break`) | `Stmt` + `showStmt` | new `xxxStmtP` + add it to `statementP` :694 | new `checkStmt` case | new `execStmt` case | ✔ |
| New keyword | – | add to `reservedWords` :315, parse with `keywordP` | | | test that `let kw = 1;` fails and `let kwx = 1;` passes |
| New `TypeError` | | | add a constructor :18–33 + a line in `showTypeError` :179 | | **add it to `typeErrorKind` :112**, or tests.kl will not compile (non-exhaustive match) |
| New `RuntimeError` | | | | constructor :33 + `showRuntimeError` :151 | |

**Watch out:** adding a constructor to `Expr`/`Stmt`/`BinOp`/`TypeError` makes every `match` on that type non-exhaustive. In pure-hm that is a **compile error** that names the file, line and missing case (`non-exhaustive match; missing case: X`). Fix each place it names: usually the printer in `syntax.kl`, `showTypeError`, and `typeErrorKind` in tests.

---

## 4. Twist recipes

Each recipe lists **file:line → what to do**, then the code, then tests. The code is copied from the tested reference folder `../minits-hope-twists/pure-hm/<name>/` and already converted to pure-hm.

### A. Modulo `%` (same precedence as `*` `/`) — `modop/`

1. `syntax.kl:32`, in `type BinOp`, add `| OpMod`:
   ```
     = OpAdd | OpSub | OpMul | OpDiv | OpMod
   ```
2. `syntax.kl:80`, in `binOpText`, after `OpDiv => "/";`, add `    OpMod => "%";`
3. `parser.kl:591`, in `multiplicationP`, replace the `orP` with a `choiceP`:
   ```
     chainl1 (unaryP expr) (choiceP [binOpP "*" OpMul, binOpP "/" OpDiv, binOpP "%" OpMod]);
   ```
4. `checker.kl:305`, in `binaryResultType`, after the `OpDiv` line:
   ```
         OpMod => if bothNumbers then Ok TNumber else bad;
   ```
5. `eval.kl:199`, in `delta2`, after the `OpDiv` line. This uses JS semantics, so the result takes the sign of the dividend: `-7 % 3 = -1`.
   ```
       OpMod =>
         numbers what a b (fn x y =>
           if y == 0.0 then Err DivisionByZero
           else Ok (VNum (x -. (y *. int_to_float (float_to_int (x /. y))))));
   ```
Tests: `17 % 5` → 2, `-7 % 3` → -1, `5.5 % 2` → 1.5, `1 % 0` → DivisionByZero, `"x" % 2` → InvalidOperands, and `a % b % c` parses as `(% (% a b) c)`.

**Variations:**
- A different operator at the *addition* level: edit `additionP` :594 instead.
- An operator at a *new* precedence level, for example `**`, tighter than `*`: write `powerP expr` between `unaryP` and `multiplicationP` and make `multiplicationP` call it.
  - **Right-associative**: `` letrec powerP expr = unaryP expr `bindP` (fn l => orP (operatorP "**" `bindP` (fn _ => mapP (fn r => EBinary OpPow l r) (powerP expr))) (pureP l)); ``
  - Also add `"*"` to the `forbidden` rules in `operatorP` :302 (`else if text == "*" then ["*"]`), so that `*` never matches the start of `**`.
  - KLang has no `pow` built-in. Compute it by repeated multiplication for whole-number exponents.

### B. `else if` — `elseif/` (sugar, parser only)

`parser.kl:666–678`, `ifStmtP`. The function must call itself, so in HM dialects change `let ifStmtP` to `letrec ifStmtP`. Then replace the last line, `` (optionP (keywordP "else" `thenP` (blockP stmt)))))))); ``, with:
```
              (optionP
                (keywordP "else" `thenP`
                  (orP
                    (blockP stmt)
                    (mapP (fn nested => SBlock [nested]) (ifStmtP stmt))))))))));
```
`else if (b) {..}` becomes `else { if (b) {..} }`.

In `tests.kl:425`, **delete** the old test `"else if is not part of the grammar"` (it would now fail). Then add: `if (a) {} else if (b) {}` parses to `(if a {} {(if b {} {})})`, a chain picks the middle branch, `if (true) {} else if (1) {}` → NonBooleanCondition.

### C. Conditional expression `c ? a : b` — `ternary/`

Precedence: looser than `||`, tighter than `=`. Right-associative.

1. `syntax.kl:48`, in `type Expr`, before `| ERecord`, add `  | ECond Expr Expr Expr`. Then in `showExpr` (after the `EField` line :158) add:
   ```
       ECond c a b => str_join ["(? ", showExpr c, " ", showExpr a, " ", showExpr b, ")"];
   ```
2. `parser.kl:614`, just above `assignmentP`, add:
   ```
   letrec conditionalP expr =
     logicalOrP expr `bindP` (fn c =>
       orP
         (symbolP "?" `thenP`
           (expr `bindP` (fn a =>
             symbolP ":" `thenP`
               (mapP (fn b => ECond c a b) (conditionalP expr)))))
         (pureP c));
   ```
   Then at `parser.kl:621` change `(logicalOrP expr);` to `(conditionalP expr);`.
3. `checker.kl:33`: change `| MissingField String Type;` to `| MissingField String Type` and add `  | BranchMismatch Type Type;`. In `showTypeError` add:
   `    BranchMismatch a b => str_join ["conditional branches have different types ", showType a, " and ", showType b];`
4. `checker.kl`, in `checkExpr`, after the `EField` case (:454–466), add:
   ```
       ECond c a b =>
         match checkExpr c env {
           Err err => Err err;
           Ok ct =>
             match ct {
               TBoolean =>
                 match checkExpr a env {
                   Err err => Err err;
                   Ok at =>
                     match checkExpr b env {
                       Err err => Err err;
                       Ok bt => if typeEq at bt then Ok at else Err (BranchMismatch at bt);
                     };
                 };
               _ => Err (NonBooleanCondition ct);
             };
         };
   ```
5. `eval.kl`, in `evalExpr`, before the `ERecord` case (:415). This evaluates **only** the chosen branch:
   ```
       ECond c a b =>
         match evalExpr c env store {
           Err err => Err err;
           Ok (Pair cv store1) =>
             match cv {
               VBool true => evalExpr a env store1;
               VBool false => evalExpr b env store1;
               _ => invalidOp "condition of ?: must be a boolean";
             };
         };
   ```
6. `tests.kl:128`, in `typeErrorKind`, add `    BranchMismatch _ _ => "BranchMismatch";`

### D. Compound assignment `+= -= *= /=` — `compound/` (sugar)

1. `parser.kl:304`, in `operatorP`'s `forbidden` list, add a line so that `+` `-` `*` `/` never match the start of `+=` and friends:
   ```
       else if (text == "+") || (text == "-") || (text == "*") || (text == "/") then ["="]
   ```
   (Put it after the `text == "="` line, before the final `else []`.)
2. `parser.kl:614`, above `assignmentP`, add:
   ```
   let compoundOpP =
     choiceP [
       mapP (fn _ => Some OpAdd) (operatorP "+="),
       mapP (fn _ => Some OpSub) (operatorP "-="),
       mapP (fn _ => Some OpMul) (operatorP "*="),
       mapP (fn _ => Some OpDiv) (operatorP "/="),
       mapP (fn _ => None) (operatorP "=")
     ];
   ```
3. `parser.kl:620`, in `assignmentP`, replace `` operatorP "=" `thenP` (mapP (fn rhs => EAssign name rhs) expr))) `` with:
   ```
         compoundOpP `bindP` (fn opOpt =>
           mapP
             (fn rhs =>
               match opOpt {
                 None => EAssign name rhs;
                 Some op => EAssign name (EBinary op (EVar name) rhs);
               })
             expr)))
   ```
`x += e` means `x = x + e`, so type checking and string `+=` come for free.

### E. `const` declarations (assigning to one is a type error) — `const/`

This one changes the type environment, so it is the largest twist.

1. `syntax.kl:55`: add `  | SConst String (Option TypeExpr) Expr`, and in `showStmt` print it like `SLet` but with `"(const "`.
2. `parser.kl:316`: add `"const"` to `reservedWords`. Above `statementP`, add a copy of `letStmtP` named `constStmtP` that uses `keywordP "const"` and builds `SConst`. Add `constStmtP,` to `statementP`.
3. `checker.kl`:
   - `:33` add `| AssignToConst String` (+ `showTypeError` line).
   - `:38–39`: each binding becomes `Pair name (Pair type isConst)`: `TypeEnv [[Pair String (Pair Type Bool)]]`.
   - `:52` `bindLocal`: keep its name and arguments (as `bindWith false`) and add `bindConst = bindWith true`.
   - `:59` `scopeLookup`: return only the type. Add `isConstVar env name`, which finds the **innermost** binding and returns its flag.
   - `:376` `EAssign`: after `Some t =>`, insert `if isConstVar env x then Err (AssignToConst x) else`.
   - Add an `SConst` case to `checkStmt`, a copy of `SLet` that calls `bindConst`.
4. `eval.kl`, `execStmt`: `SConst x ann init => execStmt (SLet x ann init) env store;`. At runtime a const behaves exactly like a let.
5. `tests.kl:128`: add `AssignToConst _ => "AssignToConst";` to `typeErrorKind`.

Copy the exact code from `../minits-hope-twists/pure-hm/const/checker.kl` lines 38–94.

### F. `for (init; cond; update) { body }` — `forloop/` (sugar)

1. `parser.kl:316`: add `"for"` to `reservedWords`.
2. `parser.kl:688`, just above `exprStmtP`:
   ```
   let forStmtP stmt =
     keywordP "for" `thenP`
       (symbolP "(" `thenP`
         (orP letStmtP (exprP `bindP` (fn e => symbolP ";" `thenP` (pureP (SExpr e)))) `bindP` (fn init =>
           exprP `bindP` (fn cond =>
             symbolP ";" `thenP`
               (exprP `bindP` (fn update =>
                 symbolP ")" `thenP`
                   (mapP
                     (fn body => SBlock [init, SWhile cond (SBlock [body, SExpr update])])
                     (blockP stmt))))))));
   ```
3. `parser.kl:698`: add `    forStmtP stmt,` to `statementP`.

The outer `SBlock` keeps `i` out of the enclosing scope. `continue` (recipe J) would skip the update in this desugaring. If both twists are asked for, run the update in the loop instead of inside the body.

### G. `do { body } while (cond);` — `dowhile/` (sugar)

1. `parser.kl:316`: add `"do"` to `reservedWords`.
2. `parser.kl:688`, above `exprStmtP`:
   ```
   let doWhileStmtP stmt =
     keywordP "do" `thenP`
       (blockP stmt `bindP` (fn body =>
         keywordP "while" `thenP`
           (symbolP "(" `thenP`
             (exprP `bindP` (fn cond =>
               symbolP ")" `thenP`
                 (symbolP ";" `thenP`
                   (pureP (SBlock [body, SWhile cond body]))))))));
   ```
3. `parser.kl:698`: add `    doWhileStmtP stmt,` after `whileStmtP stmt,`.

The body runs once, then the loop behaves like `while`.

### H. Block comments `/* ... */` — `blockcomment/` (parser only)

1. `parser.kl:281`, just above `wsP`:
   ```
   letrec skipBlockComment start inp =
     match inp {
       Input src len off _ _ =>
         if off + 1 >= len then
           PFail (errorAt start "unterminated block comment")
         else if str_slice src off (off + 2) == "*/" then
           POk unit (advanceText inp "*/") noError
         else
           skipBlockComment start (advanceInput inp (str_at src off));
     };

   let blockCommentP =
     Parser (fn inp =>
       match runP (textP "/*") inp {
         PFail e => PFail e;
         POk _ rest _ => skipBlockComment inp rest;
       });
   ```
2. `parser.kl:289`, in `wsP`: change `lineCommentP)));` to `(orP lineCommentP blockCommentP))));`.

`advanceInput` keeps line/column correct across newlines inside the comment.

### I. Comparing strings with `< <= > >=` — `strcompare/`

1. `checker.kl:306–309`: in each of the four lines change `if bothNumbers` to `if bothNumbers || bothStrings`.
2. `eval.kl:188`, above `delta2`:
   ```
   let ordering what a b numTest cmpTest =
     match a {
       VStr x => (match b { VStr y => Ok (VBool (cmpTest (str_compare x y))); _ => invalidOp what; });
       _ => numbers what a b (fn x y => Ok (VBool (numTest x y)));
     };
   ```
3. `eval.kl:200–203`: replace the four comparison lines:
   ```
       OpLt => ordering what a b (fn x y => x <. y) (fn c => c < 0);
       OpLe => ordering what a b (fn x y => x <=. y) (fn c => c <= 0);
       OpGt => ordering what a b (fn x y => x >. y) (fn c => c > 0);
       OpGe => ordering what a b (fn x y => x >=. y) (fn c => c >= 0);
   ```
4. `tests.kl:459`: **delete** the old test `"T-Compare rejects strings"`.

### J. `break;` and `continue;` — `breakcontinue/`

The idea: `break` returns an `Err` *signal* that carries the current store. Blocks, ifs, and sequences already pass `Err` up unchanged, so the signal travels until a `while` catches it. The checker uses a marker binding to know it is inside a loop.

1. `syntax.kl:59`: `| SWhile Expr Stmt` (drop the `;`), then add `  | SBreak` and `  | SContinue;`. In `showStmt` :179 add `SBreak => "(break)";` and `SContinue => "(continue)";`.
2. `parser.kl:316`: add `"break", "continue"` to `reservedWords`. Above `exprStmtP`:
   ```
   let breakStmtP = keywordP "break" `thenP` (symbolP ";" `thenP` (pureP SBreak));
   let continueStmtP = keywordP "continue" `thenP` (symbolP ";" `thenP` (pureP SContinue));
   ```
   Add `breakStmtP,` and `continueStmtP,` to `statementP` :694.
3. `checker.kl:33`: add `| OutsideLoop String` (+ `showTypeError`: `OutsideLoop what => str_join [what, " can only appear inside a loop"];`).
   - `:563` (in `SWhile`): change `checkStmt body env` to `checkStmt body (bindLocal (pushScope env) "%loop" TNull)`. `%loop` can never be a MiniTS name, so there are no clashes.
   - Add these cases to `checkStmt`:
   ```
       SBreak =>
         match lookupType env "%loop" { Some _ => Ok env; None => Err (OutsideLoop "break"); };
       SContinue =>
         match lookupType env "%loop" { Some _ => Ok env; None => Err (OutsideLoop "continue"); };
   ```
4. `eval.kl:35`: `| InvalidRuntimeOperation String` (drop the `;`), then `  | LoopBreak Store` and `  | LoopContinue Store;`. Add both to `showRuntimeError` :151.
   - `eval.kl:505–508` (`SWhile`'s `loop`): put two cases **before** `Err err => Err err;`:
   ```
                        Err (LoopBreak st2) => Ok (Pair env st2);
                        Err (LoopContinue st2) => loop st2;
   ```
   - After `loop store;` (:514), add cases to `execStmt`:
   ```
       SBreak => Err (LoopBreak store);
       SContinue => Err (LoopContinue store);
   ```
   - `eval.kl:532`, in `execProgram`, turn a stray signal into a defensive error before `Err err => Err err;`:
   ```
           Err (LoopBreak _) => Err (InvalidRuntimeOperation "break outside a loop");
           Err (LoopContinue _) => Err (InvalidRuntimeOperation "continue outside a loop");
   ```
5. `tests.kl:128`: add `OutsideLoop _ => "OutsideLoop";` to `typeErrorKind`.

### Test template (copy into `tests.kl` above `# Running`, line 652)

```
let twistTests = [
  Test "parses"            (parsesTo "src" "(expected s-expr)"),
  Test "runs"              (numIs "let a = ...;" "a" 2.0),
  Test "string result"     (strIs "let s = ...;" "s" "text"),
  Test "type error"        (typeErrorIs "..." "InvalidOperands"),
  Test "runtime error"     (divisionByZero "..."),
  Test "rejected syntax"   (parseFails "..."),
  Test "keyword reserved"  (parseFails "let kw = 1;" && parseOk "let kwx = 1;")
];
```
Then at `tests.kl:661` change `(listAppend evalTests defensiveTests))));` to `(listAppend evalTests (listAppend defensiveTests twistTests)))));`.

---

## 5. Smaller twists (sketches, not run)

These follow the same patterns but were not compiled. Use them as a starting point and test them.

- **New escape** (`\0`, `\'`): add a branch in `escapeChar` `parser.kl:385`. Also add it to `escapeText` `syntax.kl:92` if it must print back.
- **New primitive type** (`undefined`): `TyUndefined` in `syntax.kl:16`, `primitiveTypeP` :439, `reservedWords` :315, `TUndefined` in `checker.kl:9` (+ `typeEq`, `isPrimitive`, `showType`, `resolveType`), `VUndefined` in `eval.kl:11` (+ `primitiveEquals`, `showValue`), and a literal in `primaryP` :533.
- **`!=`/`==` as synonyms** of `!==`/`===`: in `equalityP` :603 add `binOpP "==" OpEq` *after* `"==="`. In `operatorP` give `"=="` the forbidden `["="]`, and remove the test `"== is not a MiniTS operator"`.
- **`&&` without short-circuit**: in `eval.kl:316` make `OpAnd`/`OpOr` evaluate both sides like the `_ =>` branch, then combine them.
- **`let` without an initializer** (defaults to `null`): in `letStmtP`, make `"=" Expr` optional and default to `ENull`. Remove the test `"let requires an initializer"`.
- **Unary `+`**: in `unaryP` :576 add `mapP (fn _ => OpPlus) (operatorP "+")` (+ `UnOp`, `unOpText`, `EUnary` case, `delta1`).
- **Count loop iterations / output log**: add a field to `Store`, for example `Store next cells log` or a counter. Most functions only pattern-match `Store next cells`, so update `allocate`, `storeGet`, `storeSet`, `storeSize`, and `emptyStore`.
- **Tests runner (pure-hm):** `report` is at `tests.kl:664` and builds an `IO Unit` with `bindIO`. `main` is at `tests.kl:677`. It prints the summary and does not set an exit status.
