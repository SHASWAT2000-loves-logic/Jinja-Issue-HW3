# Issue 2069: Jinja 3.1.6 reproduction record

## Scope and starting point

- Official repository: https://github.com/pallets/jinja
- Issue read: https://github.com/pallets/jinja/issues/2069
- Branch: `hw3-fix`; HEAD and original 3.1.6 tag commit: `15206881c006c79667fe5154fe80c01c65410679`.
- Python 3.13.15; editable local Jinja2 3.1.6; pytest 8.3.4.
- Baseline before adding tests: 909 passed.
- This step changed tests and evidence only. No implementation change, fix, or push.

## Relevant repository path

- `src/jinja2/environment.py`: `Environment.parse` starts parsing and returns a template syntax tree.
- `src/jinja2/parser.py`: `Parser.parse`, `Parser.parse_if`, and `Parser.parse_set` create template, conditional, and assignment nodes.
- `src/jinja2/nodes.py`: syntax tree node definitions used by parsing and compilation.
- `src/jinja2/meta.py`: `find_undeclared_variables` visits the tree with `TrackingCodeGenerator`; its `enter_frame` collects names that would be resolved from the context, excluding environment globals.
- `src/jinja2/compiler.py` and `src/jinja2/idtracking.py`: compilation frames and symbol tracking represent which names are loaded or assigned.
- `tests/test_api.py`: existing `TestMeta` class and `env` fixture conventions; three reproduction tests were added here.

## Issue and test expectations

The issue reports that a name assigned inside every branch of an `if`/`elif`/`else` is returned by `meta.find_undeclared_variables` when used after the conditional. A follow-up reports the same outcome for `if`/`else`. The expected undeclared set contains the condition input `control` alone because every path assigns `output` before its use. A control case assigns `output` in only one branch; then a path can use a value from the context, so both `control` and `output` should be classified as undeclared.

| Test | Expected | Actual on 3.1.6 | Result |
| --- | --- | --- | --- |
| `test_find_undeclared_variables_set_in_all_if_branches` | `{'control'}` | `{'control', 'output'}` | Failed |
| `test_find_undeclared_variables_set_in_if_and_else` | `{'control'}` | `{'control', 'output'}` | Failed |
| `test_find_undeclared_variables_set_in_only_one_if_branch` | `{'control', 'output'}` | `{'control', 'output'}` | Passed |

## Commands and evidence

Focused run:

```powershell
.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_all_if_branches tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_if_and_else tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_only_one_if_branch
```

Result: **2 failed, 1 passed** (exit code 1). The exact pytest output, including assertions and a separate passing control run, is in `evidence/issue-2069-reproduction.txt`.

## Assumptions and difficulties

- The expected classification is about names that may need a value from the template context, consistent with the `meta.find_undeclared_variables` docstring. A complete `else` makes the all-branches cases exhaustive.
- The issue describes behavior first seen in 3.1.5 and a follow-up about 3.1.4. The tests here were run only against the assigned 3.1.6 commit.
- This Windows environment previously restricted pytest temporary and cache files. The focused tests ran with the necessary filesystem permission and disabled pytest's cache provider to keep the evidence run focused. No test was skipped or marked xfail.
- No patches or solutions were consulted. Root cause analysis and implementation changes are deferred.
