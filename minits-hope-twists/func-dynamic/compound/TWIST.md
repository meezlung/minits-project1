# Twist D: Compound assignment `+= -= *= /=` (func-dynamic)

Sugar: x op= e becomes x = x op e.

- Base: `../../../minits-project1-func-dynamic` (dialect **func-dynamic**). The same twist for the other dialects: `../../pure-hm/compound/`, `../../func-hm/compound/`.
- Files changed from the base: `parser.kl`, `tests.kl`. Every changed spot is marked with a `# TWIST` comment, so Ctrl+F `TWIST` finds them all.
- Tests: `251 passed, 0 failed` with the real KLang 12.0.0 compiler. From this folder: `python -I ../../../tools/klang/klang-browser.py --run -d func-dynamic tests.kl`. The twist's own tests are in `twistTests`, just above `# Running` in `tests.kl`.
- Step-by-step guide with line numbers: recipe **D** in `../../../minits-project1-func-dynamic/HOPE-TWISTS.md`.

To see exactly what changed: `git diff --no-index ../../../minits-project1-func-dynamic .`, or open both files side by side in VS Code (*Select for Compare*).
