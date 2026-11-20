# Twist C: Conditional `c ? a : b` (func-dynamic)

New ECond expression; only the chosen branch runs; new BranchMismatch type error.

- Base: `../../../minits-project1-func-dynamic` (dialect **func-dynamic**). The same twist for the other dialects: `../../pure-hm/ternary/`, `../../func-hm/ternary/`.
- Files changed from the base: `syntax.kl`, `parser.kl`, `checker.kl`, `eval.kl`, `tests.kl`. Every changed spot is marked with a `# TWIST` comment, so Ctrl+F `TWIST` finds them all.
- Tests: `252 passed, 0 failed` with the real KLang 12.0.0 compiler. From this folder: `python -I ../../../tools/klang/klang-browser.py --run -d func-dynamic tests.kl`. The twist's own tests are in `twistTests`, just above `# Running` in `tests.kl`.
- Step-by-step guide with line numbers: recipe **C** in `../../../minits-project1-func-dynamic/HOPE-TWISTS.md`.

To see exactly what changed: `git diff --no-index ../../../minits-project1-func-dynamic .`, or open both files side by side in VS Code (*Select for Compare*).
