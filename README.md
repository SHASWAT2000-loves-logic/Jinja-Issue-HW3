# Jinja

For this repository's course project, see [Homework 3: Jinja Issue #2069](#homework-3-jinja-issue-2069) below.

Jinja is a fast, expressive, extensible templating engine. Special
placeholders in the template allow writing code similar to Python
syntax. Then the template is passed data to render the final document.

It includes:

-   Template inheritance and inclusion.
-   Define and import macros within templates.
-   HTML templates can use autoescaping to prevent XSS from untrusted
    user input.
-   A sandboxed environment can safely render untrusted templates.
-   AsyncIO support for generating templates and calling async
    functions.
-   I18N support with Babel.
-   Templates are compiled to optimized Python code just-in-time and
    cached, or can be compiled ahead-of-time.
-   Exceptions point to the correct line in templates to make debugging
    easier.
-   Extensible filters, tests, functions, and even syntax.

Jinja's philosophy is that while application logic belongs in Python if
possible, it shouldn't make the template designer's job difficult by
restricting functionality too much.


## In A Nutshell

```jinja
{% extends "base.html" %}
{% block title %}Members{% endblock %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href="{{ user.url }}">{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}
```

## Donate

The Pallets organization develops and supports Jinja and other popular
packages. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today][].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/

## Homework 3: Jinja Issue #2069

This repository is the Homework 3 submission for [Jinja Issue #2069](https://github.com/pallets/jinja/issues/2069), built from the official Jinja **3.1.6** tag. The original Jinja documentation above is retained. This section explains the assignment changes and how to verify them on Windows PowerShell.

### Bug and correction

`jinja2.meta.find_undeclared_variables()` reports names that a template may need to look up from its input context. In Jinja 3.1.6, it incorrectly reports `output` in a template that assigns `output` in every `if`/`elif`/`else` branch before using it. The expected result contains the condition input alone. If any branch leaves `output` unset, `output` must remain in the undeclared set because that path may need an input value.

The root cause was in `Symbols.branch_update()` in `src/jinja2/idtracking.py`. It merged assignments from all branches and requested a context lookup for every newly assigned name, without distinguishing assignments made on **every** path from assignments made on **some** paths. `meta.find_undeclared_variables()` observes those context-resolution actions, so the unnecessary lookup became a false positive.

The fix tracks definite assignments separately from the existing set of possible assignments. It treats each `elif` body as a separate path and includes an empty path when there is no final `else`. It avoids a context lookup only when every path assigns the name and no branch reads it first. Partial assignments, reads before assignment, and parent-scope references retain their prior behavior. This symbol tracking is also used by normal template compilation, so rendering behavior was tested alongside meta analysis.

The implementation change is confined to `src/jinja2/idtracking.py`. `tests/test_api.py` contains three original reproduction cases and 24 further validation cases covering nested and partial branches, missing `else`, reads before assignment, multiple names, globals, macros, loops, and rendered output. Investigation and test records are in [`evidence/`](evidence/):

- [`issue-2069-investigation.md`](evidence/issue-2069-investigation.md) and [`issue-2069-reproduction.txt`](evidence/issue-2069-reproduction.txt): original failure and setup observations.
- [`issue-2069-root-cause-analysis.md`](evidence/issue-2069-root-cause-analysis.md): code path, root cause, and diagnostic observations.
- [`issue-2069-patch-validation.md`](evidence/issue-2069-patch-validation.md) and [`issue-2069-test-results.txt`](evidence/issue-2069-test-results.txt): implementation reasoning and exact test results.

### Versions and test results

| Stage | Commit | Result |
| --- | --- | --- |
| Original Jinja 3.1.6 | `15206881c006c79667fe5154fe80c01c65410679` | 909 existing tests passed |
| Reproduction tests added; source still original | `a0666ac10c271281187699cc8d413eb1fad97f4e` | 2 failed, 1 passed in the focused run |
| Conditional symbol fix | `9702fc2802d49a3514d681cc1c7723d80b8896b4` | All three reproduction tests passed; full suite: 936 passed |

The commits are consecutive in the repository's Git history; no upstream history was replaced. The reproduction commit changes tests and evidence only. Its source tree can be compared with the tag using `git diff 3.1.6 a0666ac1 -- src/jinja2` (no output expected).

The recorded environment used Python **3.13.15**, pytest **8.3.4**, and an editable installation of local Jinja **3.1.6**. Jinja declares support for Python 3.7 and newer, but this homework's tests were run locally only on Python 3.13.15. Compatibility with other Python versions was not directly verified here.

### Reproduce the original bug

Run these commands from the repository root in **one PowerShell session**. They create a separate worktree and virtual environment at the reproduction commit, so the editable package cannot accidentally import the fixed checkout. Use an unused sibling directory name if `jinja-issue-2069-before` already exists. Python 3.13 must be available through the Windows `py` launcher.

```powershell
$repoRoot = (Get-Location).Path
$beforeDir = Join-Path (Split-Path -Parent $repoRoot) 'jinja-issue-2069-before'
git worktree add --detach $beforeDir a0666ac10c271281187699cc8d413eb1fad97f4e
if ($LASTEXITCODE -ne 0) { throw 'Could not create reproduction worktree' }
Set-Location $beforeDir
if ((git rev-parse HEAD) -ne 'a0666ac10c271281187699cc8d413eb1fad97f4e') { throw 'Wrong reproduction commit' }
py -3.13 -m venv .venv
if ($LASTEXITCODE -ne 0) { throw 'Could not create reproduction virtual environment' }
.\.venv\Scripts\python.exe -m pip install -r requirements\tests.txt
if ($LASTEXITCODE -ne 0) { throw 'Could not install test dependencies' }
.\.venv\Scripts\python.exe -m pip install -e .
if ($LASTEXITCODE -ne 0) { throw 'Could not install local Jinja' }
.\.venv\Scripts\python.exe -c "import pathlib, jinja2; expected = (pathlib.Path.cwd() / 'src' / 'jinja2' / '__init__.py').resolve(); actual = pathlib.Path(jinja2.__file__).resolve(); assert jinja2.__version__ == '3.1.6' and actual == expected, (jinja2.__version__, actual); print(jinja2.__version__, actual)"
if ($LASTEXITCODE -ne 0) { throw 'Jinja was not imported from the reproduction worktree' }
.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_all_if_branches tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_if_and_else tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_only_one_if_branch
if ($LASTEXITCODE -ne 1) { throw 'Expected the original reproduction tests to expose the bug' }
```

**Expected:** `2 failed, 1 passed`; pytest exits with code 1 because the two assertions intentionally expose the original bug. The import check must print a path inside `jinja-issue-2069-before\src\jinja2`. The command sequence and expected failures were independently verified in an isolated worktree. The full development lockfile, `requirements\dev.txt`, pins Jinja 3.1.4; use `requirements\tests.txt` and `pip install -e .` as shown so that the local 3.1.6 checkout is tested.

### Validate the fix

From the same PowerShell session, return to the repository root and create another separate worktree at the exact fix commit. It gets its own virtual environment. The final command is the project's complete pytest suite.

```powershell
Set-Location $repoRoot
$afterDir = Join-Path (Split-Path -Parent $repoRoot) 'jinja-issue-2069-after'
git worktree add --detach $afterDir 9702fc2802d49a3514d681cc1c7723d80b8896b4
if ($LASTEXITCODE -ne 0) { throw 'Could not create fixed worktree' }
Set-Location $afterDir
if ((git rev-parse HEAD) -ne '9702fc2802d49a3514d681cc1c7723d80b8896b4') { throw 'Wrong fix commit' }
py -3.13 -m venv .venv
if ($LASTEXITCODE -ne 0) { throw 'Could not create fixed virtual environment' }
.\.venv\Scripts\python.exe -m pip install -r requirements\tests.txt
if ($LASTEXITCODE -ne 0) { throw 'Could not install test dependencies' }
.\.venv\Scripts\python.exe -m pip install -e .
if ($LASTEXITCODE -ne 0) { throw 'Could not install local Jinja' }
.\.venv\Scripts\python.exe -c "import pathlib, jinja2; expected = (pathlib.Path.cwd() / 'src' / 'jinja2' / '__init__.py').resolve(); actual = pathlib.Path(jinja2.__file__).resolve(); assert jinja2.__version__ == '3.1.6' and actual == expected, (jinja2.__version__, actual); print(jinja2.__version__, actual)"
if ($LASTEXITCODE -ne 0) { throw 'Jinja was not imported from the fixed worktree' }
.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_all_if_branches tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_if_and_else tests/test_api.py::TestMeta::test_find_undeclared_variables_set_in_only_one_if_branch
if ($LASTEXITCODE -ne 0) { throw 'Focused fix tests failed' }
.\.venv\Scripts\python.exe -m pytest -q
if ($LASTEXITCODE -ne 0) { throw 'Full test suite failed' }
```

**Expected:** `3 passed` for the original reproduction cases and `936 passed` for the complete suite. The import check must print a path inside `jinja-issue-2069-after\src\jinja2`. Each worktree uses its own `.venv`; do not switch commits inside a worktree while reusing its editable installation.
