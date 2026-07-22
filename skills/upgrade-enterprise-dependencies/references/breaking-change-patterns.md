# Breaking-change patterns and how to resolve them

A framework major bump breaks a project in a small number of recurring *shapes*. The
specific classes, artifacts and properties differ every time; the shapes don't. This
file is the catalogue: read it before applying changes (SKILL.md §2) so you recognise a
failure instead of debugging it from scratch, and use it as the repair manual while
iterating to green (§4).

Nothing here names a version, and nothing here is a substitute for the release notes.
Each pattern is **symptom → why it happens → how to find the actual answer for *this*
upgrade**. Always confirm the answer against the target version's own documentation or
source before editing; a pattern tells you where to look, not what the replacement is.

---

## 1. A type moved to a different package

**Symptom.** `Unresolved reference` / `package … does not exist` / `cannot find symbol`
for a class whose name you still recognise, while the dependency is clearly present.

**Why.** Frameworks re-home types across majors, especially auto-configuration classes,
builders and support classes that outgrew the package they were originally filed under.
The class still exists — the import path doesn't.

**How to find the new home.**

1. Search the framework's source for the *simple* name, not the FQN — the package is
   exactly what changed:
   ```bash
   gh search code --repo <org>/<framework> --filename '<ClassName>.java' 'class <ClassName>'
   ```
   Or fetch the release/migration notes for the minor line and search them for the class
   name; large re-homings are almost always announced.
2. Failing that, resolve it from the jar that is already on the classpath — the
   authoritative answer for the version you actually pulled in:
   ```bash
   mvn -q dependency:build-classpath -Dmdep.outputFile=/dev/stdout \
     | tr ':' '\n' | grep '\.jar$' \
     | xargs -I{} sh -c 'unzip -l {} 2>/dev/null | grep -q "<ClassName>.class" && echo {}'
   unzip -l <the.jar> | grep '<ClassName>.class'   # prints the new package path
   ```
3. Only then rewrite imports. Grep the whole named module for the old package prefix,
   not just the file that failed — one move usually hits several files, and the compiler
   reports them a few at a time.

**Trap.** A moved class may also have moved *artifact* — see pattern 2. If the search
above finds no jar on the classpath containing it, you have a split, not a rename.

---

## 2. Functionality split out into a separate artifact or starter

**Symptom.** As pattern 1, but no jar on the classpath contains the class at all — or
an auto-configuration that used to apply now silently doesn't. Common when a "kitchen
sink" starter is broken into finer-grained modules so applications only pay for what
they use.

**Why.** Modularisation is a standard major-version move. The code wasn't deleted, it
was extracted into its own artifact that you must now depend on explicitly.

**How to find the new artifact.**

- The migration guide for the minor line usually lists the split explicitly ("X is now
  provided by `<new-artifact>`"). Read that first.
- Otherwise search the framework's own build for the module that contains the class
  (`gh search code`, then look at which directory/module the file lives in — the module
  directory name is normally the artifactId).
- Confirm by resolving it: add the candidate dependency **without a version** (the
  framework BOM/parent should manage it — if it doesn't, that's a sign you picked an
  artifact outside the BOM) and re-run the build.

**Fix.** Add the new dependency to the module that needs it. Prefer the framework's own
starter over a transitive coordinate, and prefer letting the BOM manage the version over
pinning one (see `compatibility.md`).

**Trap.** If the extracted piece provides AOP/proxying, instrumentation or an
auto-configuration, adding the dependency may not be enough on its own — see pattern 5.

---

## 3. A dependency is gone, replaced by a successor

**Symptom.** A managed version no longer resolves; `dependency:tree` shows the artifact
disappeared after the parent bump; or the migration guide says a library has been
removed and its capability absorbed elsewhere.

**Why.** Frameworks periodically absorb a satellite project into core, or replace a
third-party integration with a first-party one. The capability survives; the coordinates
and usually the API do not.

**How to handle it.**

1. **Identify the successor from the migration guide, not by guessing.** The guide names
   it. If it doesn't, search the framework's issue tracker for the removal
   (`gh search issues --repo <org>/<framework> '<old-artifact> removed'`).
2. **Map the API surface attribute by attribute.** A successor rarely keeps the same
   annotation attribute names, defaults, or expression support. Build an explicit
   old → new table from the successor's *source or javadoc* — not from memory, and not
   by assuming the names carried over:
   ```bash
   gh search code --repo <org>/<framework> --filename '<Annotation>.java'
   ```
   Read the annotation's declared attributes and their types. Pay particular attention
   to whether attributes that used to accept a placeholder/SpEL expression have been
   replaced by separately named `…String` variants, and whether a nested attribute
   (e.g. a backoff/retry policy object) has been flattened into top-level attributes.
3. **Check what enables it.** The old library usually had an `@Enable…` and/or its own
   starter; the successor usually has a *differently named* one, or is activated by an
   auto-configuration that requires a specific starter on the classpath. Missing this is
   the single most common way an upgrade compiles, passes, and does nothing (pattern 5).
4. **Check the runtime prerequisites.** Capabilities implemented via proxies/AOP need
   the aspect/AOP starter present. If the old library pulled it in transitively and the
   successor doesn't, add it explicitly.
5. Remove the old coordinate from dependency management *and* from every module that
   declared it, and grep the whole named module set for the old package prefix.

---

## 4. An annotation or API kept its name but changed its shape

**Symptom.** `None of the following functions can be called with the arguments supplied`
(Kotlin), `cannot find symbol: method <attr>()` (Java), or a constructor that now
demands an extra parameter.

**Why.** Same-named API, different signature — attributes renamed, narrowed to a single
type, or a required parameter added.

**How to fix.** Read the declaration in the target version (javadoc or source, per
pattern 3 step 2) and adapt the call site to what it actually declares. Do not delete
the attribute to make it compile — that silently drops configuration. If an attribute
genuinely has no successor, that behaviour has to be reproduced another way or reported
as a manual follow-up (SKILL.md §5).

**Trap in tests.** Exception and support types often gain required constructor
parameters. Where a test constructs a framework exception directly, supply the new
parameter with a test double rather than switching to a different exception type — the
assertion is about *that* type.

---

## 5. It compiles, it passes, and it does nothing

**Symptom.** None. That is the problem. Optionally a single `WARN` at startup that
nobody reads.

**Why.** Behavioural features in Spring are activated by three separate things: the
annotation at the call site, an `@Enable…`/auto-configuration that installs the
processor, and an artifact on the classpath that provides the proxying machinery. An
upgrade can move or rename any one of them independently. Fix only the import and the
code compiles perfectly while the aspect is never applied — retries don't retry, caching
doesn't cache, scheduling doesn't schedule, transactions don't demarcate.

**How to check, for every behavioural annotation touched during the upgrade.**

1. Find what installs its processor in the target version — search the framework source
   for the annotation and follow it to the `@Enable…` / registrar / auto-configuration
   that imports it.
2. Confirm that thing is actually reachable in this project: the `@Enable…` is present
   on a configuration class the application scans, or the auto-configuration's starter
   is on the classpath and its `@ConditionalOn…` guards are satisfied.
3. Confirm the machinery is present — for proxy-based features, the AOP/aspect
   dependency.
4. Make sure at least one existing test exercises the behaviour end to end (a retry test
   that counts invocations, a cache test that asserts a second call is served from the
   cache). If none exists, say so in the report rather than assuming it works.

Enabling annotations belong on a configuration/auto-configuration class the project
owns — in a library module, that means its own auto-configuration class, so consumers
get the behaviour without wiring it themselves.

---

## 6. The minimum JDK went up

**Symptom.** The framework refuses to run, or — much more often — the build fails inside
a *plugin*: `Unsupported class file major version …`, an agent that can't instrument, an
annotation processor that crashes.

**Why.** Every tool that reads or writes bytecode is pinned to a maximum class-file
version it understands. Raising the language/toolchain level therefore breaks each of
them independently of the framework upgrade itself.

**Sweep, don't spot-fix.** After raising the toolchain level, review *every* dependency
that touches bytecode and bump each to a release that supports the new class-file
version, checking each project's own release notes for the JDK it added support in:

- coverage/instrumentation agents
- bytecode-manipulation libraries and the mocking frameworks built on them
- annotation processors and the compiler plugin configuration that runs them
- shading/relocating and container-image plugins
- test runners and the surefire/failsafe generation in use
- any code generator that emits or post-processes classes

**Strong encapsulation.** Newer JDKs reject reflective access into JDK internals that
older ones merely warned about. Tools that need it (some generators, serialisation and
reflection-heavy test helpers) require explicit `--add-opens`/`--add-exports` on the
JVM that runs them — added to the *plugin's* JVM arguments (compiler, surefire, the
generator's own fork), not to the application. Add the narrowest opens that make it
pass, and record them in the report; they're a liability, not a fix to be proud of.

---

## 7. A spec baseline moved underneath you

**Symptom.** Duplicate or conflicting API classes on the classpath, `NoSuchMethodError`
at runtime against an API type, or a third-party library that no longer resolves against
the spec version the framework now manages.

**Why.** A framework major usually pins a new platform/spec baseline (validation,
annotations, persistence, servlet, XML binding). Libraries compiled against the previous
baseline may be binary-incompatible even though the coordinates resolve fine.

**How to handle.** After the bump, diff the resolved tree before and after
(`mvn dependency:tree` / `gradle dependencies`) and look at every spec API artifact that
changed. For each third-party library that integrates with one, check its own release
notes for which baseline it supports and bump it to a release built against the new one.
Grep the sources for the pre-jakarta package prefix as well — anything still importing
it is dead weight or a genuine incompatibility.

---

## 8. Configuration properties renamed, relocated or removed

**Symptom.** Context fails to load with a message naming a property; or worse, nothing
fails and a setting is silently ignored because unknown keys aren't errors.

**How to find.** Release notes list property changes per minor line; also run the
framework's configuration-metadata/migration tooling if the project already has it.
Then grep every configuration source the project actually has — `application*.yml`,
`application*.properties`, profile-specific files, test resources, container/deployment
manifests and CI environment variables — not just the main application file. A property
overridden in a deployment manifest will not fail any build.

---

## 9. Code generators emit code for the previous framework generation

**Symptom.** Generated sources don't compile, or compile against APIs that no longer
exist — API-client/server stubs, XML-binding classes, mappers, schema-derived types.

**Why.** Generators template their output against a specific framework generation and
expose that choice as a configuration flag. Bumping the framework without bumping the
generator (and re-reading *its* flags) produces output for the old generation.

**How to fix.** Bump the generator plugin to a release whose own release notes claim
support for the target framework generation, then re-read that generator's documented
options — the flag that selected the previous generation is typically superseded by a
new one rather than being inferred. Delete stale generated output before rebuilding so
you're not compiling a mixture. Generated code is not hand-editable; fixing it means
fixing the generator configuration.

---

## 10. The test harness breaks even though production code is fine

Recurring shapes, all worth checking explicitly after a framework major:

- **Framework exception/support constructors gained parameters** — pattern 4.
- **Test slices and auto-configuration imports moved** along with the production classes
  they mirror — pattern 1 applies to `@…Test` annotations and their imports too.
- **Context caching and port binding.** Tests that bind a real port fail when contexts
  are reused or when a previous test's context is still holding the port. If a random
  port is injected but arrives unset, prefer restructuring the test to read it from the
  running context over hardcoding a port; if a fixed port is genuinely necessary,
  isolate the context so it is torn down after use, and note the trade-off — a hardcoded
  port is a flake waiting for a parallel build.
- **Mocking/bytecode libraries** must support the new JDK — pattern 6.

Fixing a test is only legitimate when the test still asserts the same behaviour
(SKILL.md §5).

---

## 11. The artifact exists but the repository won't serve it

**Symptom.** `403 Forbidden` / `409` / an explicit policy message from an internal proxy
or mirror, for a coordinate that exists upstream.

**Why.** This is not a compatibility problem and must not be debugged as one. A
corporate repository proxy can block specific artifacts or version ranges by policy.

**How to handle.** First confirm it's a policy block and not a typo or a missing
repository, by checking whether *neighbouring* versions of the same coordinate resolve.
Then look for a version of the same library the proxy does serve, or an equivalent
coordinate (libraries are sometimes published under more than one groupId after a
project move). Pick the closest one that still satisfies the compatibility constraints
from `compatibility.md`. Record the substitution and the reason in the final report —
the next person will otherwise assume the odd version is arbitrary.

---

## 12. Kotlin specifics

- **Compiler plugins are versioned with the compiler.** The Spring/JPA compiler plugins
  (`all-open`, `no-arg`, and the serialization plugin if used) must match the Kotlin
  version exactly. Bump them together or the build fails in a way that looks unrelated.
- **New framework annotations need `all-open`/`no-arg` configuration.** If a framework
  major introduces or renames the annotations that mark proxied or entity classes, the
  compiler-plugin configuration listing those annotations has to be updated too, or
  Kotlin's final-by-default classes silently break proxying.
- **Nullness annotations tighten.** Frameworks adopting explicit nullness annotations
  turn what was platform-typed on the Kotlin side into strictly nullable or non-null
  types; previously fine call sites become compile errors. These are real findings, not
  noise — fix the call site, don't `!!` past it.
- **Annotation attributes are stricter in Kotlin than Java.** An attribute that widened
  or changed type upstream produces a Kotlin overload-resolution error rather than a
  clean "renamed attribute" message — read the annotation's declaration (pattern 4).
- **Field injection into a `companion object` doesn't work** the way it does on a Java
  static; if a test relies on it, restructure to instance state rather than working
  around it.
- **Strict warning settings** (`-Werror` / `allWarningsAsErrors`) turn a Kotlin bump's
  new deprecation warnings into build failures. Fix the deprecations; don't disable the
  flag to get green.

## 13. Java specifics

- **Annotation processors must be declared as processor paths**, not merely as
  dependencies, once the compiler plugin configuration is touched — a processor that is
  only a compile dependency may stop running without any error, producing
  "method does not exist" failures for generated code.
- **Processor and library versions must match the toolchain** — pattern 6.
- **`--add-opens` requirements surface at test/runtime, not compile time** — pattern 6.
