# Applying changes and verifying, one named module at a time

## Why one named module at a time, in dependency order

The reactor is a chain: root BOM → platform parent(s) → service(s). A version bump at a
parent can ripple into every module below it. Work only the modules the user named
(§1), and among those go in dependency order (parents before dependents). If you bump
several at once and build together, a failure could originate from any of them, and
you'll waste time bisecting by hand. Instead, for each named module in order:

1. Bump it only. Build just it.
2. Confirm green, commit a checkpoint.
3. Move to the next named module down.

Build only the named modules (plus whatever upstream `-am` has to install to make them
resolve — see below). **Don't run a full-reactor build** (`mvn install` from the root)
to "confirm everything" when the user scoped a subset: a parent bump breaks every
sibling that inherits it but wasn't named, so the reactor as a whole is red by design.
That red is expected and out of scope — chasing it means migrating modules the user
deliberately left for a separate request. A partial upgrade (named modules done, the
rest untouched) is a valid, safe stopping point.

## Building a `pom`-packaging module (root, platform parents)

These have no source of their own to compile, but still need verifying:

```bash
mvn -q -f <module>/pom.xml validate
mvn -q -f <module>/pom.xml install -N   # -N: this POM only, no children yet
```

`install` (not just `validate`) matters here because downstream modules resolve
these via the local repo — an uninstalled parent version means the next level's build
will fail to resolve the parent, which is a false negative unrelated to the actual
upgrade.

## Building/testing a service module

```bash
mvn -q -pl <service-module> -am compile   # -am: also build needed upstream modules
mvn -pl <service-module> -am test
```

Use `-am` so Maven builds/installs any not-yet-installed parent/dependency modules in
the same reactor automatically, rather than requiring a full `mvn install` pass from
the root every time — faster feedback loop while iterating.

## Reading build/test failures correctly

- **Compile errors** after a Spring Boot major bump are usually explained by a
  removed/renamed class the migration guide already told you about — check the
  migration brief (SKILL.md §3) before treating it as novel.
- **Bean creation / context load failures at test time** (`ApplicationContext failed
  to load`) are often a config property rename or removed autoconfiguration — grep
  `src/main/resources/application*.yml` for the property Spring's error message names.
- **Testcontainers/Kafka/DB test failures** can be caused by a transitive version
  bump (e.g. a newer Postgres driver or Confluent client) rather than by Spring Boot
  itself — check what actually changed with
  `mvn dependency:tree -Dverbose` diffed before/after.
- **A test that was already flaky/broken before this upgrade** is not this skill's
  job to fix — confirm by checking it fails identically on the pre-upgrade commit,
  then note it in the report as pre-existing rather than silently leaving it failing
  with no explanation.

## Iterate to green, don't stop at "compiles"

A module counts as done only when `mvn -pl <module> -am test` exits 0 with the real
test suite intact. Fix, rebuild, re-test, repeat. If a fix changes behavior (not just
mechanically renaming an API call), think about whether it needs a matching test
change — but the test must still assert the real behavior, not merely a relaxed or
disabled version of what it asserted before.

## Checkpoint as you go

Commit after each level goes green (see `safety.md`), so a failure three levels in
doesn't force redoing the ones that already passed.
