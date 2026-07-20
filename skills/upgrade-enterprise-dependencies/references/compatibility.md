# Finding compatible versions for coupled libraries

Spring Boot bumps its own managed versions internally, but a repo's root `pom.xml`
typically also manages, in its `<properties>`, a set of libraries Spring Boot doesn't
own — commonly Spring Cloud, Kotlin, springdoc, an ORM-integration lib (e.g.
hypersistence-utils), messaging/serialization clients (e.g. Confluent/Avro), and build
plugins (jib, jacoco, the sonar plugin), plus whatever else this particular repo
declares. Enumerate what *this* root POM actually lists rather than assuming a fixed
set. Each needs a version that's actually compatible with the target Spring Boot line,
not just "the newest version of the library."

## Priority order for each library

1. **Does Spring Boot manage it too?** After bumping the parent, check whether the
   new `spring-boot-dependencies` BOM (transitively imported via the parent) already
   manages this artifact. If so and this repo doesn't need a different version, the
   simplest and safest move is to **drop the explicit version override** and inherit
   Spring Boot's own choice — that's one less thing to keep in sync on every future
   upgrade. Check by looking at what version actually resolves
   (`mvn dependency:tree -Dincludes=<groupId>:<artifactId>` on a module that pulls it
   in) before and after removing the override.
2. **Does it publish its own Spring Boot compatibility table?** Several of these do
   (Spring Cloud most explicitly — see SKILL.md/release-notes.md; springdoc-openapi's
   README documents which release line supports which Spring Boot line). Read that
   table directly rather than inferring from semver.
3. **Otherwise, read its own release notes for the target Spring Boot/Java line.**
   Fetch the library's GitHub releases the same way as `release-notes.md` describes,
   and look specifically for "requires Spring Boot X" / "requires Java Y" / "requires
   Hibernate Z" statements — libraries that integrate with Hibernate
   (hypersistence-utils) or Kafka (Confluent/Avro) are usually pinned to a specific
   major line of their integration target, so match that first.
4. **When genuinely no compatibility statement exists**, prefer the latest version
   whose release date is close to the target Spring Boot release, and say explicitly
   in the migration brief that this pairing is inferred rather than documented — that
   honesty matters more than looking certain, since a wrong guess here fails silently
   at runtime rather than at compile time.

## Kotlin specifically

Kotlin's version is somewhat independent of Spring Boot but coupled to:
- The Kotlin Spring/JPA compiler plugins wherever they're configured (typically a
  platform-starter parent POM) — `kotlin-maven-allopen`, `kotlin-maven-noarg` — these
  must match `kotlin.version` exactly, they're not independently versioned.
- `jackson-module-kotlin` — managed by Spring Boot's Jackson BOM; check it resolves
  to a version that supports the target Kotlin version (Jackson Kotlin module
  compatibility is documented in its own changelog).
- Whatever Kotlin language/API level the code actually uses — a Kotlin bump rarely
  breaks compilation but can change `-Werror`-sensitive warnings into errors if the
  build enables strict warnings.

## Don't bump what doesn't need bumping

Not every property in `root pom.xml` needs to move just because Spring Boot did. If a
library's current version already satisfies the target Spring Boot's requirements
(check its compatibility statement per above), leave it alone — an unnecessary bump is
extra surface area for a regression with no upgrade benefit, and makes the final
report noisier and harder to review. Only touch what the compatibility check actually
requires.
