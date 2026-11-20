# Twist J: `break;` / `continue;` (pure-hm)

Err signals caught by the while loop; OutsideLoop type error.

- Base: `../../../minits-project1-pure-hm` (dialect **pure-hm**). The same twist for the other dialects: `../../func-hm/breakcontinue/`, `../../func-dynamic/breakcontinue/`.
- Files changed from the base: `syntax.kl`, `parser.kl`, `checker.kl`, `eval.kl`, `tests.kl`. Every changed spot is marked with a `# TWIST` comment, so Ctrl+F `TWIST` finds them all.
- Tests: `251 passed, 0 failed` with the real KLang 12.0.0 compiler. From this folder: `python -I ../../../tools/klang/klang-browser.py --run -d pure-hm tests.kl`. The twist's own tests are in `twistTests`, just above `# Running` in `tests.kl`.
- Step-by-step guide with line numbers: recipe **J** in `../../../minits-project1-pure-hm/HOPE-TWISTS.md`.

To see exactly what changed: `git diff --no-index ../../../minits-project1-pure-hm .`, or open both files side by side in VS Code (*Select for Compare*).
