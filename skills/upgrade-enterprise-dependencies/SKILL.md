---
name: upgrade-enterprise-dependencies
description: Upgrades Spring Boot/Framework and everything coupled to it in any Java or Kotlin project (Maven or Gradle) — the root/BOM parent, an internal platform-starter parent, or a single leaf module. Version-agnostic; no library set, module name or version is baked in. Reads every release note and migration guide in the range with the model itself (not a parser script) to find breaking changes — package moves, artifact splits, libraries replaced by a successor, annotation and property renames, minimum-JDK and Jakarta baseline raises — then rebuilds and re-tests each named module until green. Use whenever the user asks to "update Spring", "upgrade the platform", "bump the BOM", "обновить спринг/платформу/зависимости", "migrate module X", or names any library or module alongside "upgrade"/"update"/"migrate" — even without naming a module, since the skill discovers that itself. One skill, any target — don't reach for a narrower workflow because only one service or library was mentioned.
---

# Upgrade Enterprise Dependencies

Upgrades Spring and the dependency graph around it anywhere in a Java/Kotlin build —
the root/BOM parent, an internal platform-starter parent, or a single leaf module —
through the same mechanism. Nothing here is hardcoded: build tool, module topology,
current versions, target versions, the set of coupled libraries, and the breaking
changes between them are all discovered per run. A version pair this skill has never
seen is the normal case, not an edge case.

**Core rule: LLM reads, tooling orchestrates.** Release notes, changelogs, migration
guides, javadoc and compatibility docs are prose — read and reason over them directly
as text. Never write a script to parse markdown/HTML/JSON changelog content; that
throws away the nuance ("this only breaks you if you use X") that makes the fix correct
instead of mechanical. Shell/`mvn`/`gradle`/`git`/`gh` commands are fine — those are
deterministic orchestration, not interpretation.

## 0. Discover the build topology

Before touching anything, build a map of the project, because every later step depends
on it:

1. **Detect the build tool.** `pom.xml` → Maven; `build.gradle`/`build.gradle.kts` +
   `settings.gradle*` → Gradle. Both may appear in one org; use whichever the target
   modules actually use. Note where versions live — Maven `<properties>` +
   `<dependencyManagement>`, Gradle `gradle/libs.versions.toml`, `ext {}`, or a
   `dependencyManagement`/platform block.
2. **Find every build file** under the root (`find . -name pom.xml`, or read
   `settings.gradle*` for the included projects) and reconstruct the parent/platform →
   child chain from each one's `<parent>` / applied platform. A common enterprise shape
   is a root build that imports the Spring Boot parent/BOM and owns the version
   catalogue → one or more internal *platform-starter* parents (no application code) →
   individual leaf services. But always re-derive the actual chain from the build files
   every run; don't assume that shape. There may be zero, one, or several intermediate
   parents, the project may be a single module, and modules may have been added since.
3. **Classify each module** as: **BOM/framework root** (imports the Spring Boot
   parent/BOM, owns version properties), **platform parent** (internal, no app code,
   sits between the BOM and services), or **service/library** (a real module with
   sources under `src/`).
4. **Record the baseline.** Current Spring Boot version, JDK/toolchain level, Kotlin or
   Java language level, and every managed version the target modules declare. This is
   what you diff release notes against, and what the final report's version table is
   built from.

This graph gives you the order in which to apply the *named* set (§4) — a named parent
before any named child that depends on it, starting from the highest module the user
actually named, which may or may not be the root.

## 1. Resolve what the user actually wants upgraded

The user hands you the explicit set to upgrade — one module or several: just the
framework root, a platform parent, one service, several services, a standalone library
that doesn't touch the platform, or any mix (e.g. "the platform, then service X").
**Upgrade exactly the modules named — no more.** Do not widen the scope yourself:
bumping a platform parent technically affects every module that inherits it, but the
user decides which of those get migrated and when, and the ones they didn't name are
deliberately out of scope — they come in separate requests, not this one. Ask only if
the request is genuinely ambiguous (e.g. a named library matches both a
platform-managed version and a service's own override).

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
removed just now. Fetch and read **every** intermediate release note and every
per-minor-line migration guide between the current version (exclusive) and the target
(inclusive) — for Spring Boot/Framework/Cloud and third-party libraries from their
GitHub releases and wikis, and for an internal platform parent from the repo's own git
log, since it isn't published anywhere external. Full method, including exact
`gh`/`git` invocations and how to size the fetch when the range is large:
`references/release-notes.md`.

While reading, build a running list of candidate incompatibilities. `references/breaking-change-patterns.md`
is the catalogue of *shapes* these take — a class moving to a new package or a new
artifact, a starter being split up, a library dropped from the BOM in favour of a
successor with a differently-named API, a config property renamed, a minimum-JDK or
Jakarta baseline raise — with a detection recipe for each. Read it before §4 so you
recognise a failure as a known shape instead of debugging it from first principles.
Nothing in it names a version; the shapes recur across every major line.

For each candidate, check the project actually uses it (`grep -r` the class, package,
property or artifact before worrying about it) and note: what changes, why, and which
release note said so — you'll cite that in the final report.

Do the same pass for coupled libraries whose compatible version depends on the Spring
Boot line — the Spring Cloud train, Kotlin and its compiler plugins, API-doc and
ORM-integration libraries, messaging/serialisation clients, code generators, and build
plugins (coverage, image build, annotation processors, test/mocking libraries).
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
build *just that module*, confirm it resolves and installs cleanly, commit as a
checkpoint, then move to the next named module down. Never bump several named modules at
once and build them together — when it breaks, you want to know which one introduced the
problem instead of untangling errors from several changes at once.

Build only the named modules (plus whatever upstream the build tool must install to
resolve them). **Don't run a full build of every module** to "confirm everything": a
parent bump breaks every sibling that inherits it but that you weren't asked to migrate,
so the reactor as a whole is red by design when the user scoped a subset — that red is
expected and out of scope, not yours to chase.

At each step, if the build breaks, fix it using the migration brief as the map and
`references/breaking-change-patterns.md` as the repair manual — the release notes
usually say exactly what replaced what, and the catalogue says how to find the
replacement when they don't. If a failure isn't explained by anything in the brief,
treat it as a genuine surprise: read the actual compiler/test error, and go back to the
release notes you may have skimmed past. Full loop mechanics, Maven and Gradle command
equivalents, what counts as "done" for a module, and how to run the test suites:
`references/apply-and-verify.md`.

**A green compile is not proof the migration landed.** The dangerous residue of an
upgrade is the change that still compiles and still passes tests while doing nothing —
an annotation whose processor moved to a module that is no longer on the classpath, a
feature that now needs a different `@Enable…`, an auto-configuration that silently
backed off. Whenever you migrate something *behavioural* rather than purely structural,
find out what activates it in the target version and confirm the behaviour is still
exercised. `references/breaking-change-patterns.md` covers how to spot this class.

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
  for each. State plainly that verification is scoped to the named set, not a build of
  every module in the project.
- **Incompatibilities found and fixed**: symptom → root cause → fix → citing which
  release note called it out.
- **Out of scope by design**: modules that inherit a bumped parent but weren't in the
  named set — list them so the deliberately-deferred migrations are visible for the
  separate request that will handle them (a ledger, not a failure).
- **Manual follow-ups**: anything left undone and why, with enough detail that a human
  can pick it up without re-doing the research.

See `references/safety.md` for the exact git/checkpoint discipline this report
should reflect (branch name, commits made, nothing force-pushed or skipped).
