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
- Removed or default-changed autoconfiguration
- Deprecated-then-removed public API (check against what this repo's `src/` actually
  imports — `grep -r` the class/package name before worrying about it)
- Default dependency version bumps that matter (e.g. a Hibernate major bump changes
  dialect/behavior even without a compile error)
- New minimum Java version requirements
- Actuator/observability endpoint or metric name changes

Keep a running note (in your own scratch, not a file the user asked for) of
`(version, change, why it matters here, source)` as you go — this becomes the
migration brief in SKILL.md §3.

**4. Spring Cloud is a release train, not a single artifact**

`spring-cloud.version` in the root POM (a calendar-style train name) names a *train*, whose
compatible Spring Boot version is documented on the Spring Cloud release train wiki/
release notes, not inferable from the version number alone. Fetch the Spring Cloud
release notes the same way (`spring-cloud/spring-cloud-release` repo tags/releases)
and read them to confirm which train pairs with the target Spring Boot line — do not
guess from version number patterns alone, they've shifted schemes across trains.

## The platform's own history (this repo, not GitHub)

Internal platform-starter parents usually aren't published externally — their history
lives in this repo's git log. When the upgrade target is a platform version bump rather
than (or in addition to) a raw Spring Boot bump, reconstruct what changed the same way,
from git instead of GitHub — pass each platform parent's POM path to `git log`/`git diff`:

```bash
git log --oneline <current-tag-or-commit>..<target-tag-or-commit> -- \
  pom.xml <platform-parent>/pom.xml
git diff <current-tag-or-commit>..<target-tag-or-commit> -- \
  pom.xml <platform-parent>/pom.xml
```

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
