# Twist E: `const` declarations

Assigning to a const is an AssignToConst type error; TypeEnv entries carry an isConst flag.

- Base: `../../minits-project1-pure-hm` (dialect **pure-hm**).
- Files changed from the base: `syntax.kl`, `parser.kl`, `checker.kl`, `eval.kl`, `tests.kl`. Every changed spot is marked with a `# TWIST` comment, so Ctrl+F `TWIST` finds them all.
- Tests: `251 passed, 0 failed` with the real KLang 12.0.0 compiler (`python klang-browser.py --run -d pure-hm tests.kl`). The twist's own tests are in `twistTests`, just above `# Running` in `tests.kl`.
- Step-by-step guide with line numbers: recipe **E** in `HOPE-TWISTS.md` in each project folder (`../../minits-project1-<dialect>/HOPE-TWISTS.md`). That guide also gives the func-dynamic and func-hm spellings.

See the changes yourself: `git diff --no-index ../../minits-project1-pure-hm const` (run from this folder's parent), or open both files side by side in VS Code (*Select for Compare*).
