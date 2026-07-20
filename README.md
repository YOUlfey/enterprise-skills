# enterprise-skills

Agent skills for enterprise Spring/Maven upgrade work, installable via [skills.sh](https://skills.sh).

## Skills

- **[upgrade-enterprise-dependencies](skills/upgrade-enterprise-dependencies/)** — safely
  upgrade Spring (Boot/Cloud) and coupled dependency versions anywhere in a Maven reactor
  (root BOM, an internal platform-starter parent, or a leaf service). It reads *every*
  release note across the version range with the model itself (not a parser script) to
  find breaking changes, applies only the modules you name in dependency order, and
  rebuilds/re-tests each until green.

## Install

```bash
npx skills add YOUlfey/enterprise-skills
```
