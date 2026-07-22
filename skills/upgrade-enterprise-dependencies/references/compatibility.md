# Finding compatible versions for coupled libraries

Spring Boot bumps its own managed versions internally, but a project's root build almost
always also manages a set of libraries Spring Boot doesn't own — commonly a Spring Cloud
train, the Kotlin toolchain, an API-documentation library, an ORM-integration library,
messaging/serialisation clients, code generators, and build plugins (coverage, container
image, static analysis), plus whatever else this particular project declares. Enumerate
what *this* build actually lists — Maven `<properties>`/`<dependencyManagement>`, or a
Gradle version catalogue/platform — rather than assuming a fixed set. Each needs a
version that is actually compatible with the target Spring Boot line *and* the target
JDK level, not just "the newest version of the library".

## Priority order for each library

1. **Does Spring Boot manage it now?** After bumping the parent/BOM, check whether the
   new `spring-boot-dependencies` BOM already manages this artifact. If so, and this
   project doesn't need a different version, the simplest and safest move is to **drop
   the explicit version override** and inherit Spring Boot's own choice — one less thing
   to keep in sync on every future upgrade. Verify by looking at what actually resolves
   (`mvn dependency:tree -Dincludes=<groupId>:<artifactId>`, or
   `gradle dependencyInsight --dependency <artifactId>`) before and after removing the
   override.
2. **Does it publish its own Spring Boot compatibility table?** Several do — the Spring
   Cloud release train most explicitly, and API-doc/integration libraries usually
   document which of their release lines supports which Spring Boot line. Read that
   table directly rather than inferring from semver.
3. **Otherwise, read its own release notes for the target Spring Boot and JDK lines.**
   Fetch the library's GitHub releases the way `release-notes.md` describes, and look
   specifically for "requires Spring Boot X" / "requires Java Y" / "requires
   <integration target> Z" statements. Libraries that integrate tightly with something
   else (an ORM, a broker client, a bytecode agent) are usually pinned to a specific
   major line of their integration target — match that first, then the framework.
4. **When genuinely no compatibility statement exists**, prefer the latest release whose
   date is close to the target Spring Boot release, and say explicitly in the migration
   brief that this pairing is inferred rather than documented — that honesty matters more
   than looking certain, since a wrong guess here tends to fail at runtime rather than at
   compile time.

## When the coordinate itself changed

A library that was renamed, donated to a foundation, or split up may be published under
a new groupId or split into several artifacts, so "the newest version" of the old
coordinate is a dead end that looks like a stalled project. If a library's latest release
is suspiciously old relative to the framework you're targeting, check whether development
moved — the old artifact's README or last release note normally says where.
`breaking-change-patterns.md` §2 and §3 cover the code-side consequences.

## Kotlin specifically

Kotlin's version is somewhat independent of Spring Boot but coupled to:

- **The Kotlin compiler plugins**, wherever they're configured (typically a platform
  parent or a convention plugin) — `all-open`, `no-arg`, serialization. These must match
  the Kotlin version exactly; they are not independently versioned.
- **The Jackson Kotlin module** — normally managed by Spring Boot's Jackson BOM; check
  that it resolves to a release supporting the target Kotlin version, and note that a
  Jackson major can also change its own groupIds.
- **The Kotlin language/API level the code actually targets** — a Kotlin bump rarely
  breaks compilation outright, but it does turn new deprecation warnings into errors when
  the build treats warnings as errors.

## Don't bump what doesn't need bumping

Not every managed version needs to move just because Spring Boot did. If a library's
current version already satisfies the target Spring Boot's and JDK's requirements (check
its compatibility statement per above), leave it alone — an unnecessary bump is extra
surface area for a regression with no upgrade benefit, and makes the final report noisier
and harder to review. Only touch what the compatibility check actually requires.
