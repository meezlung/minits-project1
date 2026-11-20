# Possible HOPE twists: reference implementations

Layout: `minits-hope-twists/<dialect>/<twist>/`. Each folder is a complete copy of `../minits-project1-<dialect>` with one twist added. All 30 (10 twists × 3 dialects) were compiled and run with the real KLang 12.0.0 compiler (`../tools/klang/klang-browser.py`), and every test passes, including the 242 original tests.

| Recipe | Twist | Folders | Files changed (besides tests) | Tests (each dialect) |
|---|---|---|---|---|
| A | Modulo `%` | [pure-hm](pure-hm/modop/TWIST.md) · [func-hm](func-hm/modop/TWIST.md) · [func-dynamic](func-dynamic/modop/TWIST.md) | syntax, parser, checker, eval | 249 |
| B | `else if` | [pure-hm](pure-hm/elseif/TWIST.md) · [func-hm](func-hm/elseif/TWIST.md) · [func-dynamic](func-dynamic/elseif/TWIST.md) | parser | 247 |
| C | Conditional `c ? a : b` | [pure-hm](pure-hm/ternary/TWIST.md) · [func-hm](func-hm/ternary/TWIST.md) · [func-dynamic](func-dynamic/ternary/TWIST.md) | syntax, parser, checker, eval | 252 |
| D | Compound assignment `+= -= *= /=` | [pure-hm](pure-hm/compound/TWIST.md) · [func-hm](func-hm/compound/TWIST.md) · [func-dynamic](func-dynamic/compound/TWIST.md) | parser | 251 |
| E | `const` declarations | [pure-hm](pure-hm/const/TWIST.md) · [func-hm](func-hm/const/TWIST.md) · [func-dynamic](func-dynamic/const/TWIST.md) | syntax, parser, checker, eval | 251 |
| F | `for (init; cond; update) {..}` | [pure-hm](pure-hm/forloop/TWIST.md) · [func-hm](func-hm/forloop/TWIST.md) · [func-dynamic](func-dynamic/forloop/TWIST.md) | parser | 250 |
| G | `do {..} while (c);` | [pure-hm](pure-hm/dowhile/TWIST.md) · [func-hm](func-hm/dowhile/TWIST.md) · [func-dynamic](func-dynamic/dowhile/TWIST.md) | parser | 248 |
| H | Block comments `/* */` | [pure-hm](pure-hm/blockcomment/TWIST.md) · [func-hm](func-hm/blockcomment/TWIST.md) · [func-dynamic](func-dynamic/blockcomment/TWIST.md) | parser | 250 |
| I | String `< <= > >=` | [pure-hm](pure-hm/strcompare/TWIST.md) · [func-hm](func-hm/strcompare/TWIST.md) · [func-dynamic](func-dynamic/strcompare/TWIST.md) | checker, eval | 247 |
| J | `break;` / `continue;` | [pure-hm](pure-hm/breakcontinue/TWIST.md) · [func-hm](func-hm/breakcontinue/TWIST.md) · [func-dynamic](func-dynamic/breakcontinue/TWIST.md) | syntax, parser, checker, eval | 251 |

The three dialect versions of a twist contain the same interpreter code, apart from the dialect's spelling:
- func-hm's source files are identical to pure-hm's.
- func-dynamic writes `let` for `letrec` and `[]` for `unit`.
- Only `tests.kl` differs in how it prints results.

How to use these during the HOPE:

1. Read recipe *X* in `../minits-project1-<your dialect>/HOPE-TWISTS.md`. It gives `file:line` edit points for your own folder, with the code already in your dialect.
2. Compare against `<your dialect>/<twist>/` here to check your edit. Every change is marked `# TWIST`.
3. Add the twist's tests from that folder's `tests.kl` (`twistTests`) and rerun.

The twists are independent. Each one starts from the unmodified base, so doing two twists means applying both sets of edits. Recipe F (`for`) notes one interaction, with `continue`.
