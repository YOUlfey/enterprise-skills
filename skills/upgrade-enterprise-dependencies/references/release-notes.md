# Gathering release notes: full range, LLM-read

The goal of this step is not "get the diff between two versions" — it's "know
everything that changed at every step in between," because a breaking change
introduced two minor versions ago and only *removed* in the target version won't show
up if you only diff endpoints. That means fetching a list, then reading each entry.

## External projects (Spring Boot, Spring Cloud, and other GitHub-hosted deps)

Use `gh` (when it's available and authenticated) over hitting GitHub's web UI — it
returns clean text/JSON without needing HTML scraping.

**1. List versions in range**

```bash
gh release list --repo spring-projects/spring-boot --limit 200 \
  --exclude-drafts --exclude-pre-releases
```

Filter the output yourself to the range `(current, target]`. If the user wants an RC
included, drop `--exclude-pre-releases`.

**2. Fetch every release note body in that range**

For each version in range:

```bash
gh release view v<version> --repo spring-projects/spring-boot --json body,tagName,name
```

Do this for *every* tag in range, not just the first and last — small releases in
between often carry the actual "X is now removed" notice that a big minor bump's own
note doesn't repeat. Batch these calls (they're independent — fire them without
waiting on each other one at a time when your tool supports parallel calls).

If the release body is terse and points at a milestone/changelog page instead of
listing changes inline (Spring Boot's release notes often link to a GitHub milestone
or a wiki "Release Notes" page for the minor line), follow that link with `WebFetch`
and read the actual content — don't stop at the release body if it's just a pointer.

**3. Read them yourself — don't summarize-then-discard**

Read each note's actual text. You're looking for, roughly in order of how often they
bite:
- Removed/renamed configuration properties (`spring.*` keys)
- Removed, relocated or default-changed auto-configuration
- Classes moved to a different package, or functionality extracted into a separate
  artifact/starter that now has to be depended on explicitly
- A managed dependency dropped in favour of a first-party successor, and the API
  differences that come with it
- Deprecated-then-removed public API (check against what the project's sources actually
  import — `grep -r` the class/package name before worrying about it)
- Default dependency version bumps that matter (an ORM major bump changes
  dialect/behaviour even without a compile error)
- New minimum Java version, or a new platform/spec baseline
- Actuator/observability endpoint or metric name changes

`breaking-change-patterns.md` describes each of these shapes and how to resolve it —
read it alongside the notes so you know what a given sentence in a changelog will cost.

Keep a running note (in your own scratch, not a file the user asked for) of
`(version, change, why it matters here, source)` as you go — this becomes the
migration brief in SKILL.md §3.

**4. Spring Cloud is a release train, not a single artifact**

The Spring Cloud version declared in the root build (a calendar-style train name) names
a *train*, whose compatible Spring Boot version is documented on the Spring Cloud release
train wiki/release notes and is not inferable from the version number alone. Fetch those
release notes the same way (`spring-cloud/spring-cloud-release` repo tags/releases) and
read them to confirm which train pairs with the target Spring Boot line — do not guess
from version-number patterns, they've shifted schemes across trains.

The same three-step method (list releases in range → fetch each body → read it) applies
to any GitHub-hosted third-party dependency you have to bump; only the repo changes.

## An internal platform's own history (git, not GitHub releases)

Internal platform-starter parents usually aren't published externally — their history
lives in the repo's git log. When the upgrade target is a platform version bump rather
than (or in addition to) a raw Spring Boot bump, reconstruct what changed the same way,
from git instead of GitHub — pass the root and each platform parent's build file to
`git log`/`git diff`:

```bash
git log --oneline <current-tag-or-commit>..<target-tag-or-commit> -- \
  <root-build-file> <platform-parent>/<build-file>
git diff <current-tag-or-commit>..<target-tag-or-commit> -- \
  <root-build-file> <platform-parent>/<build-file>
```

If the platform is consumed as a published artifact rather than as a sibling module in
the same repo, its sources may not be here at all. In that case get the change set from
wherever it *is* published — the artifact's own repository, its changelog, or by diffing
the two resolved dependency trees (`mvn dependency:tree` on a module pinned to the old
version versus the new one), which at minimum tells you every managed version that moved.

Read every commit in the range (not just the final diff) the same way you'd read
release notes — a commit that bumped a version and then a later commit that reverted
it matters differently than one that stuck. If there are no tags yet (a fresh repo),
treat the current HEAD as the baseline and there's nothing historical to gather yet —
note that in the report instead of fabricating history.

## Sizing the fetch

If the range spans many releases (e.g. a jump across a major boundary spanning dozens of
releases), don't skip minor releases to save time — instead parallelize the fetches
(they're read-only and independent) and prioritize reading the *migration guide* for
each major/minor line first (Spring Boot publishes one per minor version,
`https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-X.Y-Release-Notes`
style pages) before the patch-level release notes, since migration guides are written
specifically to be complete for that purpose.
