# Issue 2069: root cause analysis on Jinja 3.1.6

## Provenance and scope

- Base tag `3.1.6`: `15206881c006c79667fe5154fe80c01c65410679`.
- Reproduction-stage commit: `a0666ac10c271281187699cc8d413eb1fad97f4e` (`test: reproduce Jinja issue 2069`). Its parent is the base commit.
- The reproduction commit contains only `tests/test_api.py` and the two Part 1 evidence files. No implementation changes were made for this analysis.
- Diagnostics below ran the local editable 3.1.6 package using `.venv/Scripts/python.exe`. They were read-only and did not use patches, mocks, or newer Jinja versions.

## Execution flow and relevant locations

1. `Environment.parse` in `src/jinja2/environment.py:599-623` calls `Parser.parse`, which returns a `nodes.Template` (`src/jinja2/parser.py:1045-1049`). `Parser.parse_if` (`parser.py:255-271`) fills an `If` node's test, body, `elif_`, and `else_`; `Parser.parse_set` (`parser.py:225-234`) creates an `Assign` node.
2. The AST node layout is defined in `src/jinja2/nodes.py`: `Template` at 294-300, `Output` at 303-309, `If` at 337-344, `Assign` at 450-455, and `Name` at 539-550. A `Name` has `ctx="load"` for a read and `ctx="store"` for an assignment target. The complete two-branch example produced `[(control, load), (output, store), (output, store), (output, load)]`; both branches and the final read are present in the AST.
3. `find_undeclared_variables` in `src/jinja2/meta.py:34-55` creates `TrackingCodeGenerator` and visits the AST. `CodeGenerator.visit_Template` in `src/jinja2/compiler.py:873-893` creates a root `Frame`, calls `frame.symbols.analyze_node(node)` at line 885, and enters that frame at line 890. The later `CodeGenerator.visit_If` at 1320-1339 emits the conditional using a soft frame; the name classification has already been calculated for the root frame.
4. `Frame` owns a `Symbols` object (`compiler.py:165-196`). `Symbols.analyze_node` (`src/jinja2/idtracking.py:49-51`) uses `RootVisitor`, whose template visitor delegates to `FrameSymbolVisitor` (`idtracking.py:173-187`). `FrameSymbolVisitor.visit_Name` (`idtracking.py:238-247`) sends assignment targets to `Symbols.store` and reads to `Symbols.load`. `visit_Assign` (`idtracking.py:286-289`) visits the value expression before the target, preserving a read that occurs before assignment.
5. `Symbols.store` (`idtracking.py:97-113`) records the name in `stores` and, for a new name, creates a local reference with an `undefined` initial load. `Symbols.load` (`idtracking.py:119-121`) creates a `resolve` load for a name without a reference. `refs` maps a template name to its generated local name, such as `output -> l_0_output`; `loads` maps that local name to an action and parameter, such as `('resolve', 'output')`.
6. `FrameSymbolVisitor.visit_If` (`idtracking.py:252-268`) visits the condition, copies the symbols for the body, `elif_`, and `else_`, and merges them using `Symbols.branch_update` (`idtracking.py:123-145`). An absent `else` is represented by an empty path. A parsed `elif` is itself an `If` node in the `elif_` list.
7. `CodeGenerator.enter_frame` (`compiler.py:584-599`) emits a context lookup for every `resolve` load. `TrackingCodeGenerator.enter_frame` (`meta.py:25-31`) examines the same loads and adds each `resolve` parameter not in environment globals to `undeclared_identifiers`. `find_undeclared_variables` returns that set. Thus a symbol marked `resolve` becomes an undeclared result even if a later assignment overwrites the generated local on every path.

## Direct diagnostic observations

I parsed each template with `Environment.parse`, analyzed the AST with `Symbols.analyze_node`, inspected branch copies using `FrameSymbolVisitor`, called `meta.find_undeclared_variables`, and inspected `Environment.compile(..., raw=True)`. No code was modified. The first three cases correspond to the Part 1 tests.

| Case | Branch `stores` before outer merge: if / elif / else | Merged `output` load | Actual meta result | Expected meta result |
| --- | --- | --- | --- | --- |
| A: assignment in both `if` and `else` | `['output'] / [] / ['output']` | `('resolve', 'output')` | `{'control', 'output'}` | `{'control'}` |
| B: assignment in `if` only; `else` has no assignment | `['output'] / [] / []` | `('resolve', 'output')` | `{'control', 'output'}` | `{'control', 'output'}` |
| C: assignment in `if`, `elif`, and `else` | `['output'] / ['output'] / ['output']` | `('resolve', 'output')` | `{'control', 'output'}` | `{'control'}` |

In A, the `if` and `else` branch copies both begin with `output_load=('undefined', None)`, whereas the empty `elif` copy has no `output` entry. In B, only the `if` copy stores `output`. In C, the `elif` copy already has `('resolve', 'output')` because it is represented as an `If` node with its own empty `else_`; the outer merge still ends at `resolve`. Before the merge, the root had no `output` reference in all three cases. After the merge and after visiting the final `{{ output }}`, it had `stores={'output'}` and `output_load=('resolve', 'output')` in all three.

For A, B, and C, compiled output contains both `l_0_control = resolve('control')` and `l_0_output = resolve('output')`. Runtime behavior distinguishes the cases: with `control=False, output=99`, A renders `2`, whereas B renders `none99`. A's context lookup for `output` is unnecessary; B can use the external value. The focused reproduction tests reported **2 failed, 1 passed**; the separate control run passed. The exact pytest output is in `evidence/issue-2069-reproduction.txt`.

## Root cause and hypothesis check

The incorrect classification is introduced in `Symbols.branch_update`, especially `src/jinja2/idtracking.py:123-145`. It collects the **union** of names stored by any branch (lines 124-129), merges branch references and load actions (131-134), then unconditionally sets each newly stored name's load to `VAR_LOAD_RESOLVE` at line 145 unless a parent-scope alias applies. It does not distinguish a name stored in every executable path from one stored in only some paths. In both cases it preloads the name from context at frame entry. The tracking code generator then reports that preload as an undeclared variable. This is a symbol-tracking decision with a visible code-generation consequence, rather than a malformed AST or an independent rule in `meta.py`.

The initial alternatives were checked against the code and diagnostics:

- **Parser omitted an assignment or final `else`: unsupported.** The AST contains both stores in A and all three stores in C, and `else_present=True` in each complete case.
- **`meta.py` independently treats any assignment as undeclared: unsupported.** It only collects `resolve` loads from symbols; the wrong `resolve` action already exists after `branch_update`.
- **Branch merging loses definite-assignment information: supported.** A and B have different branch-store sets but identical merged loads; C is likewise marked `resolve` despite all three branches storing `output`.

There was no failed diagnostic hypothesis or change of implementation direction. The key distinction is that the current result is the **same** for complete and incomplete assignments, although the expected results differ.

## Closely related cases checked

| Scenario | Actual 3.1.6 meta result | Interpretation |
| --- | --- | --- |
| Nested exhaustive `if`/`else` assigning `output` | `{'a', 'b', 'output'}` | Same false positive can propagate through nested branch merges. |
| `output` assigned before an `if`, then possibly reassigned | `{'control'}` | Existing prior store prevents a new context-resolution classification. |
| `output` used before a later exhaustive conditional assignment | `{'control', 'output'}` | Correct: the earlier read may need context. |
| `if` assignment with no `else`, followed by `{{ output }}` | `{'control', 'output'}` | Correct: the false path may need context. |
| Exhaustive branches, but the `if` assignment reads old `output` on its right-hand side | `{'control', 'output'}` | Correct: an external read can occur before that branch's store. |
| Complete or incomplete conditional assignment with no later read of `output` | `{'control', 'output'}` in both | The eager resolve can also be reported when the name is only assigned. |

These observations show why a correction cannot simply remove every name that occurs in an assignment. It must preserve a real context read before assignment or along any executable path lacking an assignment. The nested and no-`else` cases also need explicit validation in Part 3.

## Conceptual correction strategy; no code changed

Make the conditional symbol merge distinguish names definitely assigned on every possible path from names only possibly assigned. Treat an absent final `else` as a path with no assignment, account for `elif` as part of one exhaustive chain, and preserve a `resolve` action when a value is read before its assignment. Only avoid the context-resolution preload when the name cannot be read from context on any path. The change should be made consistently for symbol tracking and generated code, then validated with the existing reproduction tests and the related cases above.

## Difficulties and limits

- The Part 1 pytest log contains literal trailing spaces from pytest's failure formatting. `git diff --check` flagged those lines during the reproduction commit; the log was preserved verbatim as evidence.
- Windows sandbox restrictions affected temporary pytest files during setup and reproduction. The read-only diagnostic scripts here completed without a permission issue.
- This investigation used the assigned 3.1.6 source only. It did not examine external fixes, run a full test suite, or implement a correction.
