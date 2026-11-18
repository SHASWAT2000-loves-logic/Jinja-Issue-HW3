# Issue 2069: patch and validation record

## Provenance

- Original Jinja 3.1.6 tag commit: `15206881c006c79667fe5154fe80c01c65410679`.
- Separate reproduction commit: `a0666ac10c271281187699cc8d413eb1fad97f4e` (`test: reproduce Jinja issue 2069`).
- Root-cause analysis: `evidence/issue-2069-root-cause-analysis.md`.
- This patch was derived from the local 3.1.6 implementation and the Part 1 reproduction tests. No external patch or newer-version fix was consulted.

## Implementation and reasoning

The investigation found that `Symbols.branch_update` retained the union of names assigned in branches and always gave each newly stored name a `resolve` load. That produced an unnecessary context lookup when every branch assigned the name. The `elif_` field is a list of `If` nodes; visiting it as one nested conditional also introduced a synthetic no-assignment path.

`src/jinja2/idtracking.py` now tracks `definite_stores` alongside the existing `stores`. Each assignment marks a name definite at that point; symbol copies carry this state. At a conditional merge, the intersection of definite stores across **actual** paths determines names assigned regardless of which branch runs. The union of ordinary `stores` remains in place for existing compiler behavior. The merge keeps a `resolve` load for names read before assignment in any branch and preserves parent-scope aliases. It uses an `undefined` initial load only when a newly stored name is definite across all paths and no earlier context read requires resolution.

`FrameSymbolVisitor.visit_If` now analyzes each `elif` test before any branch body and treats each `elif` body as a separate path. The final `else_` body is another path; when there is no `else`, its empty path prevents a conditional assignment from being treated as definite. Nested conditionals carry their definite-store result into their enclosing branch.

An initial draft used the assignment-expression syntax (`:=`) while collecting branch resolves. It was removed before running tests because Jinja 3.1.6 supports Python 3.7 and that syntax requires Python 3.8. No post-patch test failed, and no test assertion from the reproduction commit was weakened or removed.

## Validation coverage and expected behavior

The three original Part 1 tests remain unchanged: complete `if`/`elif`/`else` and `if`/`else` assignments must report only the condition input; a partial assignment must also report `output`. They all pass after the patch.

The 24 new cases in `tests/test_api.py` use complete-set equality for meta analysis and exact-output equality for rendering. They cover:

1. A partial `if`/`elif`/`else` chain, and an `if` without `else`: the unassigned path still needs an external value.
2. Exhaustive and incomplete nested conditionals: only the incomplete one needs external `output`.
3. Assignment before a conditional, and a read before assignment: earlier assignment avoids a context lookup; earlier read retains one.
4. Multiple names where `always` is definite and `maybe` is conditional: only `maybe` remains undeclared alongside the condition.
5. Reads in an assignment value and in an `elif` condition: both retain the required external lookup.
6. A declared environment global, a macro-local conditional assignment, a loop-local assignment, and a loop assignment that does not escape the loop.
7. Rendering through complete, partial, nested, `elif`, macro, and loop paths, including an external fallback and preservation of an outer value after the loop.

The original 909-test baseline, three reproduction tests, and 24 new cases account for the final total of 936 tests.

## Commands and results

The complete console output and exit codes are preserved in `evidence/issue-2069-test-results.txt`.

| Stage | Command | Result |
| --- | --- | --- |
| Original baseline, before Part 1 | `.\.venv\Scripts\python.exe -m pytest -q --tb=short --basetemp=.venv\pytest-basetemp -p no:cacheprovider` | 909 passed |
| Original reproduction, before patch | Focused three tests, recorded in `evidence/issue-2069-reproduction.txt` | 2 failed, 1 passed (expected bug reproduction) |
| Original reproduction, after patch | `.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider` plus the three explicit Part 1 test node IDs | 3 passed; exit 0 |
| Focused validation | `.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider tests/test_api.py::TestMeta tests/test_idtracking.py tests/test_core_tags.py` | 98 passed; exit 0 |
| Full regression | `.\.venv\Scripts\python.exe -m pytest -q` | 936 passed in 2.65s; exit 0 |

The full run reported no failures, errors, skips, or warnings. The project config treats warnings as errors. Existing tests continued to pass, including symbol-tracking and rendering tests. The full run used filesystem permission needed for pytest's temporary and cache files in this Windows environment; the command itself was unchanged.

## Review and limits

- The implementation changed only `src/jinja2/idtracking.py`. Test additions are confined to `tests/test_api.py`. The reproduction tests and earlier evidence remain present.
- The code uses syntax compatible with Jinja's declared Python 3.7 minimum. Execution was validated locally on Python 3.13.15; other Python versions were not available here.
- The patch addresses definite assignment across conditional branches while preserving Jinja's existing eager fallback for partial assignments. A separate sequence in which a partial conditional assignment is always overwritten by a later unconditional assignment can still retain an unnecessary preload; that behavior is outside the reported exhaustive-branch case and is also represented by the existing symbol-tracking expectations for partial assignments.
- No push, README work, or PDF reflection work was performed.
