# Safety discipline

An upgrade that "works" but destroys the ability to see what changed or roll back
isn't actually safe, no matter how green the tests are. Set this up *before* the
first edit, not after something goes wrong.

## Before touching anything

1. **Confirm a git repo exists.** Check with `git status`. If this directory isn't a
   git repo yet, don't silently proceed editing ungoverned files — tell the user and
   offer to `git init` (mirroring how other risky-operation workflows in this
   environment handle a missing repo). Without version control there is no safe
   rollback path, so get one in place first.
2. **Check for uncommitted work.** `git status` again, specifically for anything
   already dirty. If there's pre-existing uncommitted work, stash it (`git stash -u`)
   or ask the user what to do with it — never start editing on top of someone else's
   in-progress changes without accounting for them first.
3. **Create a dedicated branch** for the upgrade (e.g.
   `upgrade/spring-boot-<target-version>` or `upgrade/<library>-<target-version>`) —
   never commit upgrade work directly to the trunk branch. This also means the whole
   upgrade is trivially abandonable: if it goes badly, the user can just not merge the
   branch, with zero cleanup.

## While working

- **Commit at each checkpoint** described in `apply-and-verify.md` — after each named
  module goes green, not only at the very end. A commit message that says which
  module and what version bump (e.g. "root: spring-boot-starter-parent
  <old> → <new>") makes the eventual PR/report trivial to write and makes a bisect
  possible if something surfaces later.
- **Never force-push, never rewrite history that's already been shared.** This
  matters even mid-task — if you need to redo a checkpoint commit, make a new commit
  fixing it forward rather than amending and force-pushing over something already
  pushed.
- **Never skip hooks** (`--no-verify` or equivalent) to get a commit or push through.
  If a hook fails, that's signal, not friction — investigate it like any other build
  failure.
- **Don't push automatically.** Land the branch locally with every named module green
  — if the user scoped a subset, sibling modules that inherit a bumped parent may not
  build, and that's expected, not a regression to fix — then tell the user it's ready
  and let them decide whether/when to push or open a PR — pushing and opening PRs are
  visible-to-others actions that warrant an explicit go rather than being bundled into
  "upgrade the dependencies."

## Never fake a pass

Covered in SKILL.md §5, worth restating as a safety rule specifically: deleting,
disabling, or weakening a test to make a red build go green is not a fix, it's
hiding a regression from everyone who trusts the test suite after you. If a genuine
fix isn't achievable in this pass, leave the failure red, explain why in the report,
and hand it back as a manual follow-up rather than making it invisible.

## What the final report must let the user verify

Per SKILL.md §6, but concretely: the user should be able to check out the branch and
independently confirm your claims — `git log` shows one commit per checkpoint with
clear messages, `git diff <base>..<branch>` shows exactly the version/config/code
changes, and re-running `mvn test` — scoped the same way you verified it, `-pl <module>
-am`, not a full-reactor build — on any module you upgraded reproduces the green result
you reported. If any of those three wouldn't hold up, the report isn't done yet.
