# HW3 investigation and validation

## Part 1. Reproduction

The checkout preserves the original Jinja 3.1.6 release commit
`15206881c006c79667fe5154fe80c01c65410679` and its signed tag object.
The shell could not reach GitHub directly. The release files, tree, commit
and tag were retrieved through the GitHub API and checked by Git object ID.
Local history is shallow at the release commit.

The reporter sets `output` in an `if`, an `elif`, and an `else`, then prints
it. Only the guard reads `control` from the input context.
[Issue 2069](https://github.com/pallets/jinja/issues/2069)

Commit `da81cb6` adds exactly one pytest case on top of the release. The
following output was captured before the first source edit.

```text
F                                                                        [100%]
=================================== FAILURES ===================================
______________ TestMeta.test_conditional_assignment_all_branches _______________

self = <test_api.TestMeta object at 0x10940b980>
env = <jinja2.environment.Environment object at 0x10940b200>

    def test_conditional_assignment_all_branches(self, env):
        ast = env.parse(
            "{% if control == 'a' %}{% set output = 1 %}"
            "{% elif control == 'b' %}{% set output = 2 %}"
            "{% else %}{% set output = 3 %}{% endif %}{{ output }}"
        )
>       assert meta.find_undeclared_variables(ast) == {"control"}
E       AssertionError: assert {'control', 'output'} == {'control'}
E
E         Extra items in the left set:
E         'output'
E         Use -v to get more diff

tests/test_api.py:188: AssertionError
=========================== short test summary info ============================
FAILED tests/test_api.py::TestMeta::test_conditional_assignment_all_branches
1 failed in 0.03s
```

## Part 2. Diagnosis

1. `Environment.parse` constructs the template AST. In `parser.py`,
   `Parser.parse_if` represents the initial body, the list of `elif`
   nodes, and the final else body separately.
2. In `meta.py`, `find_undeclared_variables` runs a tracking code generator.
   Its `enter_frame` method records names whose symbol-load instruction is
   `resolve`, excluding environment globals.
3. On 3.1.6, the symbol table contained `('resolve', 'x')` for both a
   complete conditional assigning `x` on all paths and an incomplete
   conditional assigning it on one path. In `idtracking.py`,
   `FrameSymbolVisitor.visit_If` groups all elif nodes into one analysis.
   `Symbols.branch_update` takes the union of new stores and assigns each
   an outer alias or context lookup. It loses the distinction between
   possible and guaranteed initialization. The metadata API reports the
   lookup requested by the compiler.

## Part 3. Fix and validation

Commit `e989db9` analyzes each arm separately and includes an empty
fallthrough when there is no else. An incoming value can be omitted when
every arm initializes the variable locally before reading it, and an
earlier nested scope does not need its value. Each variable is checked
independently. The metadata API itself needs no change.

The reproducer and additional tests exercise `if/elif/else`, empty or
missing arms, nested conditionals, mixed assignments, guards and right-hand
sides that read the incoming value, parent and child scopes, capture filters,
macro defaults and extension overlay mappings. Tests check both dependency
sets and rendered output where losing the incoming value would change it.

| Clean checkout | Reproducer | Full pytest suite |
| --- | --- | --- |
| `da81cb6`, original code plus reproducer | 1 failed in 0.03s | 1 failed, 909 passed in 0.83s |
| `e989db9`, patch and final tests | 1 passed in 0.02s | 927 passed in 0.75s |

Two independent reviewers exercised generated templates beyond the committed
suite. One checked 729 three-arm combinations and compared 5,832 rendered
outputs with 3.1.6. The other checked 2,401 four-arm combinations and 9,604
render paths against expected behavior. Both reported zero mismatches on the
final implementation. These exploratory checks are separate from the pytest
counts above.

The independent review caught cases the initial suite missed. For example,
a capture filter using an incoming value should render `S1`; an intermediate
patch rendered only `1`. Another intermediate patch lost the external
`kwargs` used as a macro default. The final suite contains regressions for
both. A synthetic 800-scope memory check also identified repeated copies of
the visible store set. Sharing first-store positions brought measured peak
symbol-analysis memory from 14.77 MiB to approximately 0.33 MiB.

**Limits.** Testing used CPython 3.12.14. Static analysis remains conservative
about when macros run. For example, a macro defined before the assignment but
invoked only afterward can still cause the variable to appear undeclared.

### Reproduce the two revisions locally

From the repository root, create an environment and two detached worktrees.
These commands select the source tree explicitly for each pytest run.
The first pytest invocation is expected to fail.

```sh
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements/tests.txt MarkupSafe Babel
git worktree add --detach ../hw3-before da81cb6
git worktree add --detach ../hw3-after e989db9
(cd ../hw3-before && PYTHONPATH=src python -m pytest -q tests/test_api.py::TestMeta::test_conditional_assignment_all_branches)
(cd ../hw3-after && PYTHONPATH=src python -m pytest -q tests/test_api.py::TestMeta::test_conditional_assignment_all_branches)
(cd ../hw3-after && PYTHONPATH=src python -m pytest -q)
```

The recorded runs used pytest 9.1.1 and Trio 0.27.0. The installation command
above follows the repository's pinned test requirements, which specify pytest
8.3.4. Pytest output and elapsed times can vary by environment.
