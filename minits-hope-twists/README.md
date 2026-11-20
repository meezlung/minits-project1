# Possible HOPE twists: reference implementations

Each folder is a complete copy of `../minits-project1-pure-hm` with one twist added. All of them were compiled and run with the real KLang 12.0.0 compiler, and every test passes, including the 242 original tests.

| Folder | Recipe | Twist | Files changed (besides tests) | Tests |
|---|---|---|---|---|
| [modop/](modop/TWIST.md) | A | Modulo `%` | syntax, parser, checker, eval | 249/249 |
| [elseif/](elseif/TWIST.md) | B | `else if` | parser | 247/247 |
| [ternary/](ternary/TWIST.md) | C | Conditional `c ? a : b` | syntax, parser, checker, eval | 252/252 |
| [compound/](compound/TWIST.md) | D | Compound assignment `+= -= *= /=` | parser | 251/251 |
| [const/](const/TWIST.md) | E | `const` declarations | syntax, parser, checker, eval | 251/251 |
| [forloop/](forloop/TWIST.md) | F | `for (init; cond; update) {..}` | parser | 250/250 |
| [dowhile/](dowhile/TWIST.md) | G | `do {..} while (c);` | parser | 248/248 |
| [blockcomment/](blockcomment/TWIST.md) | H | Block comments `/* */` | parser | 250/250 |
| [strcompare/](strcompare/TWIST.md) | I | String `< <= > >=` | checker, eval | 247/247 |
| [breakcontinue/](breakcontinue/TWIST.md) | J | `break;` / `continue;` | syntax, parser, checker, eval | 251/251 |

How to use these during the HOPE:

1. Read recipe *X* in `../minits-project1-<your dialect>/HOPE-TWISTS.md`. It gives file:line edit points for your own folder, with the code already converted to your dialect.
2. Compare against the folder here to check your edit. Every change is marked `# TWIST`.
3. Add the twist's tests from that folder's `tests.kl` (`twistTests`) and rerun.

The twists are independent. Each one starts from the unmodified base, so doing two twists means applying both sets of edits. Recipe F (`for`) notes one interaction, with `continue`.
