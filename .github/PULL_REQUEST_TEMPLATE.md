<!-- Thanks for contributing! Keep PRs focused and small. -->

## What & why

<!-- What does this change, and what problem/issue does it address? -->

Closes #

## How it was tested

<!-- Commands you ran; new/updated tests. CI runs ruff + mypy + pytest. -->

- [ ] `ruff check . && ruff format --check .`
- [ ] `mypy knighteval`
- [ ] `pytest`

## Checklist

- [ ] Focused change; conventional-commit title (`feat:`/`fix:`/`docs:`/…).
- [ ] Tests added/updated for the behavior.
- [ ] Docs updated if user-facing (README / DESIGN / examples).
- [ ] **No secrets** in code, tests, fixtures, or saved-run output.
- [ ] Preserved the layering rule: `engine`/`evaluators`/`scoring` do not import `cli`/`reports`.

## Stable contracts

- [ ] This PR does **not** change a [stable contract](../blob/master/DESIGN.md#15).
- [ ] It does — and I've flagged it here and updated the docs/tests that pin it:

<!-- describe the contract change and migration impact, if any -->
