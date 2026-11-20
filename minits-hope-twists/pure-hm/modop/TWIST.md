# Twist A: Modulo `%` (pure-hm)

% at the same precedence as * and /, JS sign rules, % 0 is DivisionByZero.

- Base: `../../../minits-project1-pure-hm` (dialect **pure-hm**). The same twist for the other dialects: `../../func-hm/modop/`, `../../func-dynamic/modop/`.
- Files changed from the base: `syntax.kl`, `parser.kl`, `checker.kl`, `eval.kl`, `tests.kl`. Every changed spot is marked with a `# TWIST` comment, so Ctrl+F `TWIST` finds them all.
- Tests: `249 passed, 0 failed` with the real KLang 12.0.0 compiler. From this folder: `python -I ../../../tools/klang/klang-browser.py --run -d pure-hm tests.kl`. The twist's own tests are in `twistTests`, just above `# Running` in `tests.kl`.
- Step-by-step guide with line numbers: recipe **A** in `../../../minits-project1-pure-hm/HOPE-TWISTS.md`.

To see exactly what changed: `git diff --no-index ../../../minits-project1-pure-hm .`, or open both files side by side in VS Code (*Select for Compare*).
