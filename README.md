# enterprise-skills

Agent skills for enterprise Spring/Maven upgrade work, installable via [skills.sh](https://skills.sh).

## Skills

- **[upgrade-enterprise-dependencies](skills/upgrade-enterprise-dependencies/)** — safely
  upgrade Spring (Boot/Framework/Cloud) and everything coupled to it in any Java or Kotlin
  project built with Maven or Gradle — the root/BOM parent, an internal platform-starter
  parent, or a single leaf module. It reads *every* release note and migration guide across
  the version range with the model itself (not a parser script), recognises breaking changes
  by shape rather than by version (package moves, artifact splits, libraries replaced by a
  successor, annotations that compile but silently no-op, minimum-JDK and spec-baseline
  raises), applies only the modules you name in dependency order, and rebuilds/re-tests each
  until green. Nothing about versions, module names or library sets is baked in.

## Install

```bash
npx skills add YOUlfey/enterprise-skills
```
