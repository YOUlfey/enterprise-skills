# Applying changes and verifying, one named module at a time

## Why one named module at a time, in dependency order

The build is a chain: root/BOM → platform parent(s) → service or library modules. A
version bump at a parent can ripple into every module below it. Work only the modules
the user named (SKILL.md §1), and among those go in dependency order (parents before
dependents). If you bump several at once and build together, a failure could originate
from any of them, and you'll waste time bisecting by hand. Instead, for each named
module in order:

1. Bump it only. Build just it.
2. Confirm green, commit a checkpoint.
3. Move to the next named module down.

Build only the named modules (plus whatever upstream has to be built to make them
resolve — see below). **Don't build every module in the project** to "confirm
everything" when the user scoped a subset: a parent bump breaks every sibling that
inherits it but wasn't named, so the build as a whole is red by design. That red is
expected and out of scope — chasing it means migrating modules the user deliberately
left for a separate request. A partial upgrade (named modules done, the rest untouched)
is a valid, safe stopping point.

## Maven

**A `pom`-packaging module (root, platform parents)** has no source of its own to
compile, but still needs verifying:

```bash
mvn -q -f <module>/pom.xml validate
mvn -q -f <module>/pom.xml install -N   # -N: this POM only, no children yet
```

`install` (not just `validate`) matters here because downstream modules resolve these
via the local repository — an uninstalled parent version means the next level's build
fails to resolve the parent, a false negative unrelated to the actual upgrade.

**A module with sources:**

```bash
mvn -q -pl <module> -am compile   # -am: also build needed upstream modules
mvn -pl <module> -am test
```

Use `-am` so Maven builds any not-yet-installed parent/dependency module in the same
reactor automatically, rather than requiring a full `install` pass from the root every
time — faster feedback while iterating.

## Gradle

The same discipline, different commands. Build one project at a time via its path; the
task graph pulls in the project dependencies it needs, so there's no `-am` equivalent to
add:

```bash
./gradlew :<project-path>:compileKotlin   # or :compileJava
./gradlew :<project-path>:test
```

For a platform/BOM project (`java-platform` or a convention plugin), there is nothing to
compile — verify that dependents still resolve:

```bash
./gradlew :<dependent-project>:dependencies --configuration runtimeClasspath
```

Avoid a bare `./gradlew build` at the root for the same reason as a full Maven reactor
build: it drags in modules that were deliberately left out of scope. Prefer
`--no-build-cache` when a result looks suspiciously green after a version change.

## Reading build/test failures correctly

- **Compile errors** after a framework major bump are usually a known shape — a moved
  class, an extracted artifact, a renamed annotation attribute. Check the migration brief
  (SKILL.md §3) and `breaking-change-patterns.md` before treating a failure as novel.
- **Bean creation / context load failures at test time** (`ApplicationContext failed to
  load`) are often a config property rename or a removed/relocated auto-configuration —
  grep every configuration source for the property the error names, not just the main
  application file.
- **Container/database/messaging test failures** can be caused by a transitive version
  bump (a newer driver or client) rather than by the framework itself — check what
  actually changed by diffing the resolved dependency tree before and after.
- **Failures inside a build plugin rather than in your code** — an agent, processor or
  generator — usually mean a toolchain-level raise, not an API break. See
  `breaking-change-patterns.md` §6.
- **A test that was already flaky/broken before this upgrade** is not this skill's job to
  fix — confirm it fails identically on the pre-upgrade commit, then note it in the
  report as pre-existing rather than silently leaving it failing with no explanation.

## Iterate to green, don't stop at "compiles"

A module counts as done only when its test task exits 0 with the real test suite intact.
Fix, rebuild, re-test, repeat. If a fix changes behavior (not just mechanically renaming
an API call), think about whether it needs a matching test change — but the test must
still assert the real behavior, not a relaxed or disabled version of what it asserted
before. And a passing suite does not prove a *behavioural* migration landed — see
SKILL.md §4 and `breaking-change-patterns.md` §5.

## Checkpoint as you go

Commit after each level goes green (see `safety.md`), so a failure three levels in
doesn't force redoing the ones that already passed.
