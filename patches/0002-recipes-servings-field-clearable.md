# 0002 — Recipes: the servings field can be cleared to type a new count

Patch: `0002-recipes-servings-field-clearable.patch` (against `main` @ `c79a86b3`)

## Problem

On the create/edit recipe form, **Servings** starts at `1`. Deleting the `1` to type `2` doesn't
work: the field refills with `1` the moment it is empty. The only ways to get to 2 were to type
`12` and then delete the `1`, select the whole value and type over it, or use the arrow keys.

## Cause

`apps/web/app/(app)/recipes/edit/components/recipe-form.tsx:702`:

```tsx
onChange={(value) => setServings(parseInt(value, 10) || 1)}
```

An empty field gives `""`, and `parseInt("")` is `NaN`, so `|| 1` sets the state back to 1. The
field is controlled (`value={servings.toString()}`), so the `1` reappears straight away. Because
of this, the existing `servingsMin` validation could never fire for an empty field.

## Fix

- `servings` state is `number | null`, and `null` means the field is empty. The nutrition fields
  in the same form already use this pattern
  (`value ? parseInt(value, 10) || null : null`, shown as `""` when null).
- `validate()` refuses `null` with the existing `recipes.validation.servingsMin` message
  ("Servings must be at least 1"), so no new translation strings are needed.
- `handleSubmit` adds `|| servings === null` to its early return. `validate()` already refuses
  an empty field; the extra check narrows the type to `number` for the payload.
- `use-recipe-form-dirty-state.ts`: `RecipeFormState.servings` is `number | null` to match.
  An emptied field counts as a change.

Edge case: typing `0` empties the field (`0 || null`), the same as the nutrition fields. `min={1}`
already marks 0 as invalid.

## Tests

New file `apps/web/__tests__/app/recipes/recipe-form-servings.test.tsx`. The form's heavy
sections (media, tags, time inputs, provenance) are stubbed. The ingredient and step inputs are
stubbed to a button that fills them with one line, so the form can save.

1. the field can be cleared (empty, not `1`) and `2` typed in;
2. clearing and typing `2` saves `servings: 2`;
3. saving while it is empty shows `servingsMin` and does not call `createRecipe`.

What I ran (in a clean worktree of `main` with the patch applied):

| Check                                         | Result                                                       |
| --------------------------------------------- | ------------------------------------------------------------ |
| New tests without the fix                     | 1 and 3 fail; 2 passes (it guards the save path)             |
| New tests with the fix                        | 3/3 pass                                                     |
| Whole `apps/web` Vitest suite                 | 166 files, 1220 tests pass                                   |
| `eslint` on the changed source files          | no new warnings (`recipe-form.tsx` already has 11 on `main`) |
| `prettier --check`, `tsc --noEmit` (apps/web) | clean                                                        |

Not run: `pnpm build`, `pnpm test:e2e`, a manual check in a real browser. No E2E suite covers
recipe creation. The parked `apps/mobile` app has its own form and is not touched.

## Apply and commit

```bash
git switch -c fix/recipe-servings-clearable
git apply patches/0002-recipes-servings-field-clearable.patch
git add apps/web
git commit   # paste the message below
```

## Commit message

```
fix(recipes): the servings field can be cleared to type a new count

The servings field parsed every change with `parseInt(value, 10) || 1`,
so clearing it to type a new count put the 1 straight back. Going from
1 to 2 meant typing 12 and deleting the 1.

Servings is now null while the field is empty, the way the nutrition
fields already are. Saving with it empty is refused with the existing
"at least 1" message instead of being impossible to reach.

Co-Authored-By: Claude <noreply@anthropic.com>
```

## Issue to open first

CONTRIBUTING.md: *"PRs without a linked issue will be closed."* No existing issue covers this.
Suggested issue:

> **Title:** Recipe form: servings field can't be cleared, so changing 1 to 2 needs a workaround
>
> When adding a recipe, **Servings** defaults to 1. Deleting the 1 to type another number doesn't
> work: the field immediately puts the 1 back. To get 2 I have to type 12 and then remove the 1.

## PR description (follows `.github/pull_request_template.md`)

```markdown
## Description

The recipe form's servings field refilled itself with `1` whenever it was emptied
(`parseInt(value, 10) || 1`), so a count couldn't be replaced by deleting it and typing a new
one. Servings is now `null` while the field is empty, matching the nutrition fields in the same
form. Saving with it empty shows the existing "Servings must be at least 1" error.

## Related Issue(s)

Fixes #<issue>

## Type of Change

- [x] Bug fix (non-breaking change that fixes an issue)

## How Has This Been Tested?

- [x] Unit tests: new `recipe-form-servings.test.tsx` (clear and retype, save the new count, refuse empty)
- [x] Manual testing: <fill in after trying it in the browser>

## Checklist

- [x] My code follows the style guidelines of this project
- [x] I have performed a self-review of my own code
- [x] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation (release note: see below)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Release note

As with 0001: the next release-notes page doesn't exist yet (editable docs are still labelled
`0.25.1-beta`, already shipped), so offer this bullet in the PR for **Fixes and Improvements**:

> - **Recipe form:** The servings field can now be cleared to type a new number, instead of snapping back to 1.
