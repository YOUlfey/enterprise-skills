---
name: upgrade-enterprise-dependencies
description: Safely upgrades Spring (Boot/Cloud) and its cascading dependency versions anywhere in this Maven reactor — the root BOM/parent POM, an intermediate platform-starter parent, or any single downstream service — reading every release note in the version range with the LLM itself (not a parser script) to find breaking changes, then rebuilding and re-testing until green. Use this whenever the user asks to "update Spring", "upgrade the platform", "bump the BOM", "обновить спринг/платформу/BOM/зависимости", "migrate service X to the new platform version", or names any library/module in this repo alongside "upgrade"/"update"/"migrate" — even if they don't say which module, since this skill figures that out itself. One skill, any target: don't reach for a narrower workflow just because the request only mentions one service or one library.
---

# Upgrade Enterprise Dependencies

Upgrades Spring and its dependency graph anywhere in this Maven reactor — root BOM, a
platform-starter parent, or a single leaf service — through the same mechanism. The
target is never hardcoded: discover it from the repo and the user's request each time.

**Core rule: LLM reads, Python orchestrates.** Release notes, changelogs, migration
guides, and compatibility docs are prose — read and reason over them directly as text.
Never write a script to parse markdown/HTML/JSON changelog content; that throws away
the nuance ("this only breaks you if you use X") that makes the fix correct instead of
mechanical. Shell/`mvn`/`git`/`gh` commands are fine — those are deterministic
orchestration, not interpretation.

## 0. Discover the reactor topology

Before touching anything, build a mental map of the repo, because every later step
depends on it:

1. Find every `pom.xml` under the repo root (`find . -name pom.xml`).
2. For each, read its `<parent>` block to reconstruct the parent → child chain. A
   common enterprise shape is root `pom.xml` (imports `spring-boot-starter-parent` +
   owns the `<dependencyManagement>` BOM) → one or more internal *platform-starter*
   parents (`pom`-packaging, no app code) → individual leaf services — but always
   re-derive the actual chain from the POMs every run; don't assume this shape. There
   may be zero, one, or several intermediate parents, and new services may have been
   added.
3. Classify each POM as one of:
   - **BOM/framework root** — imports the Spring Boot parent and owns version
     properties (`<properties>` + `<dependencyManagement>`).
   - **Platform parent** — an internal `pom` packaging module with no app code, sitting
     between the BOM and services.
   - **Service** — a `jar`/leaf module with actual source under `src/`.

This graph gives you the dependency order to apply the *named* set in later (§4): a
named parent before any named child that depends on it — starting from the highest
module the user actually named, which may or may not be the root.

## 1. Resolve what the user actually wants upgraded

The user hands you the explicit set to upgrade — one module or several: just the
framework root, a platform parent, one service, several services, a standalone library
that doesn't touch the platform, or any mix (e.g. "the platform, then service X").
**Upgrade exactly the modules named — no more.** Do not widen
the scope yourself: bumping a platform parent technically affects every module that
inherits it, but the user decides which of those get migrated and when, and the ones
they didn't name are deliberately out of scope — they come in separate requests, not
this one. Ask only if the request is genuinely ambiguous (e.g. a named library matches
both a platform-managed version and a service's own override).

Two things still hold no matter how narrow the set is:

- **Order the named set by dependency, not by the order the user listed it**: a named
  parent (root/platform) is bumped and built before any named child that depends on it
  (§4). A module the user didn't name is never bumped or built here, even if it sits
  between two that they did.
- **Read upstream context even for what you won't modify** (§2): if the user names a
  service but not the platform it inherits, still read what changed in that
  platform/framework line — the service's own fixes have to match it.

Each named item resolves to its own **target version**: an explicit version/tag the
user gave, or otherwise the latest stable release of *that* artifact (skip
`-RC`/`-M`/`-SNAPSHOT` unless asked). Latest-stable is the default even when it lands on
a new major — staying current on a security-supported line is the point — so don't stop
to ask the user to pick between a patch and a major. A standalone library named on its
own is computed to *its* latest the same way, from *its* release notes, not the
framework's. See `references/release-notes.md` for looking versions up and
`references/compatibility.md` for sizing a library bump against the Spring Boot line.

## 2. Read every release note in the range — not just the endpoints

Breaking changes accumulate release by release; diffing only current→target and
reading the last changelog misses things that were deprecated three versions ago and
removed just now. Fetch and read **every** intermediate release note between the
current version (exclusive) and the target (inclusive) — for Spring Boot/Cloud from
GitHub, and for the platform's own internal history from this repo's git log (the
platform isn't published anywhere external). Full method, including exact `gh`/`git`
invocations and how to size the fetch when the range is large: `references/release-notes.md`.

While reading, build a running list of candidate incompatibilities — anything that
mentions a class, property, or dependency this repo actually uses (check with `grep`
before worrying about it). For each, note: what changes, why, and which release note
said so (you'll cite this in the final report).

Do the same pass for coupled libraries whose compatible version depends on the Spring
Boot line — Spring Cloud train, springdoc-openapi, hypersistence-utils, Kotlin, the
Spring Boot Maven/Jib/Jacoco plugins, testing libs (mockk/springmockk/wiremock).
`references/compatibility.md` explains how to find the right paired version for each
and how confident to be when no official matrix exists.

## 3. Write the migration brief before editing anything

Turn the notes from §2 into a short brief: version bumps table (dependency → old →
new), and a bulleted list of expected incompatibilities, each with its source
citation. Share this with the user only if the change is large/surprising enough to
warrant a pause — but sharing the brief is informational: the target is already
resolved to latest-stable (§1), so don't turn it into a "which version do you want?"
question. Otherwise proceed, since the user asked you to do the upgrade, not just plan
it. Keep the brief; it becomes the backbone of the final report and lets you
recognize a build/test failure as "the thing release note X warned about" instead of
debugging it from scratch.

## 4. Apply changes and rebuild, one named module at a time, in dependency order

Set up safety first (branch, working-tree check) — see `references/safety.md`, do
this before any edit.

Then, walking the **named** set in dependency order (§0): bump the highest named module
first — e.g. the framework root's parent version and managed dependency versions —
build *just that module* (`mvn -q -N validate` / `install` on the parent POM), confirm
it installs cleanly, commit as a checkpoint, then move to the next named module down.
Never bump several named modules at once and build them together — when it breaks, you
want to know which one introduced the problem instead of untangling errors from several
changes at once.

Build only the named modules (plus whatever upstream `-am` must install to resolve
them). **Don't run a full-reactor build** (`mvn install` from the root) to "confirm
everything": a parent bump breaks every sibling that inherits it but that you weren't
asked to migrate, so the reactor as a whole is red by design when the user scoped a
subset — that red is expected and out of scope, not yours to chase.

At each step, if the build breaks, fix it using the migration brief as the map (the
release notes usually say exactly what replaced what). If a failure isn't explained
by anything in the brief, treat it as a genuine surprise — read the actual compiler/
test error, and if needed go back and search the release notes you may have skimmed
past. Full loop mechanics, what counts as "done" for a module, and how to run the
service test suites: `references/apply-and-verify.md`.

## 5. Never fake green

A module is only done when it builds *and* its real test suite passes — not when
tests are deleted, `@Disabled`, weakened, or replaced with something that no longer
exercises the behavior. If a test fails for a reason unrelated to this upgrade
(pre-existing flake), say so explicitly in the report rather than silently touching
it. If something is genuinely infeasible to fix automatically (e.g. a paid migration
tool, a manual data migration, a decision only a human can make), leave it failing,
say why, and list it as a manual follow-up — do not mask it.

## 6. Final report

Always end with:
- **Version table**: every dependency/plugin bumped, old → new.
- **Per-module build status**: which named modules were upgraded, build/test result
  for each. State plainly that verification is scoped to the named set (built with
  `-pl <module> -am`), not a full-reactor build.
- **Incompatibilities found and fixed**: symptom → root cause → fix → citing which
  release note called it out.
- **Out of scope by design**: modules that inherit a bumped parent but weren't in the
  named set — list them so the deliberately-deferred migrations are visible for the
  separate request that will handle them (a ledger, not a failure).
- **Manual follow-ups**: anything left undone and why, with enough detail that a human
  can pick it up without re-doing the research.

See `references/safety.md` for the exact git/checkpoint discipline this report
should reflect (branch name, commits made, nothing force-pushed or skipped).
