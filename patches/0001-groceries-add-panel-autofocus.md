# 0001 — Groceries: focus the name field when the add panel opens

Patch: `0001-groceries-add-panel-autofocus.patch` (against `main` @ `c79a86b3`)

## Problem

On the groceries page, tapping **Add items** (or the desktop header's add button) opens the
add-grocery panel, but the name field is not focused. You have to tap the field before you
can type, every time.

## Cause

`apps/web/components/Panel/consumers/add-grocery-panel.tsx:123`: the name `<Input>` has no
`autoFocus`, and the `Panel` (a vaul drawer) does not focus anything itself. vaul 1.1.2 calls
`preventDefault()` on Radix's `onOpenAutoFocus` unless `autoFocus` is set on the drawer root,
and turning that on wouldn't help: Radix would focus the first tabbable element, which is the
panel's close button.

## Fix

Add `autoFocus` to the name input, with the `jsx-a11y/no-autofocus` disable. The store editor
panel already does this inside the same `Panel` (`components/groceries/stores/store-editor-panel.tsx:89`).

This works with the drawer because:

- React focuses the input during commit, before Radix's `FocusScope` mount effect runs.
- `FocusScope` skips its own mount auto-focus when the container already holds focus.
- Radix `Presence` unmounts the drawer content when the drawer closes, so the input mounts
  again, and takes focus again, on every open.

## Tests

Added `AddGroceryPanel, opening it` to
`apps/web/__tests__/components/groceries/grocery-panels.test.tsx`. It checks that the name
field has focus on open, and again after a close and reopen.

What I ran (in a clean worktree of `main` with the patch applied):

| Check                                         | Result                     |
| --------------------------------------------- | -------------------------- |
| New test without the fix                      | fails (field not focused)  |
| New test with the fix                         | passes; file 16/16         |
| Whole `apps/web` Vitest suite                 | 166 files, 1220 tests pass |
| `eslint` on the changed source file           | no new warnings            |
| `prettier --check`, `tsc --noEmit` (apps/web) | clean                      |

Not run: `pnpm build`, `pnpm test:e2e`, a manual check in a real browser. No E2E suite covers this
panel. **Worth checking by hand on a phone before opening the PR.** iOS Safari only raises the
keyboard when focus happens inside the tap's gesture. I expect it to, because the panel mounts in
the same commit as the tap's state update, but I couldn't verify that on a device.

Not included (possible follow-up): after you press **Add**, focus stays on the button. The pantry
view puts focus back in its field after each add (`components/groceries/pantry/pantry-view.tsx:104`).
The add panel stays open for batch adding, so it could do the same.

## Apply and commit

```bash
git switch -c fix/grocery-add-panel-autofocus
git apply patches/0001-groceries-add-panel-autofocus.patch
git add apps/web
git commit   # paste the message below
```

## Commit message

```
fix(groceries): the add panel puts the cursor in the name field

Opening the add-grocery panel did not move focus into it, so adding an
item took a tap on the name field before any typing. The name field now
takes focus when the panel opens.

vaul suppresses Radix's open auto-focus, and Radix's FocusScope leaves
an element that already holds focus alone, so autoFocus on the input is
enough, the way the store editor panel does it. The drawer mounts its
content afresh on every open, so this applies each time, not only the
first.

Co-Authored-By: Claude <noreply@anthropic.com>
```

## Issue to open first

CONTRIBUTING.md: *"PRs without a linked issue will be closed."* No existing issue covers this.
Suggested issue:

> **Title:** Add-grocery panel: name field is not focused when the panel opens
>
> When I press **Add items** on the groceries page, the panel opens but the name field has no
> focus. I have to tap it before I can type. Since the panel exists to type a grocery, the field
> should be focused when it opens.

## PR description (follows `.github/pull_request_template.md`)

```markdown
## Description

The add-grocery panel now focuses its name field when it opens, so you can type the grocery
straight away. It uses `autoFocus` on the input, the same way the store editor panel does inside
the same `Panel`. vaul suppresses Radix's open auto-focus, and Radix's `FocusScope` keeps focus
that is already inside it, so nothing moves it afterwards.

## Related Issue(s)

Fixes #<issue>

## Type of Change

- [x] Bug fix (non-breaking change that fixes an issue)

## How Has This Been Tested?

- [x] Unit tests: `grocery-panels.test.tsx` checks the field is focused on open and on reopen
- [x] Manual testing: <fill in after checking on desktop and a phone>

## Checklist

- [x] My code follows the style guidelines of this project
- [x] I have performed a self-review of my own code
- [x] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation (release note: see below)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Release note

`docs/agents/feature-docs.md` asks for a bullet under **Fixes and Improvements** on the Target
Version's release-notes page. That page doesn't exist yet: the editable docs are still labelled
`0.25.1-beta`, which has already shipped. Creating it means choosing the next version
(`pnpm docs_update <target>` in `apps/docs`), so leave that to the maintainers and offer this
text in the PR:

> - **Groceries:** The add panel now puts the cursor in the name field, so you can start typing straight away.
