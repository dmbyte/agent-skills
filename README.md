# agent-skills

Bare-metal / BMC operational skills and skill corrections gathered while
bringing up an HPE ProLiant DL360p Gen8 (iLO4 v2.82) with openSUSE Leap 16.0.
Verified against real hardware (host dl360-1, iLO4 v2.82 Feb 06 2023).
No credentials are included in this repository.

## Contents

- skills/ilo4-html5-console/SKILL.md - new skill: driving iLO4 through the
  HTML5 KVM broker (re-login after session expiry, console launch, virtual
  media over HTTP, Serial-over-LAN via SM-CLP) plus the input-reliability
  limits that decide which console channel to use.
- skills/bmc-html5-kvm-addendum/README.md - proposed corrections to the
  installed bmc-html5-kvm skill, which documents iLO5 only.

## Intended remote

https://github.com/dmbyte/agent-skills

## How to publish (credentials required, currently absent here)

```bash
cd /project/agent-skills
git remote add origin git@github.com/dmbyte/agent-skills.git
git push -u origin main
```

This environment has no gh CLI, no GITHUB_TOKEN and no SSH key, so no push
was attempted; anonymous ls-remote reports "Repository not found".
