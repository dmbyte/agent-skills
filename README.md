# agent-skills

Bare-metal / BMC operational skills and skill corrections gathered while
bringing up an HPE ProLiant DL360p Gen8 (iLO4 v2.82) with openSUSE Leap 16.0.
Verified against real HPE ProLiant Gen8 hardware (iLO4 v2.82).
No credentials are included in this repository.

## Contents

- skills/ilo4-html5-console/SKILL.md - new skill: driving iLO4 through the
  HTML5 KVM broker (re-login after session expiry, console launch, virtual
  media over HTTP, Serial-over-LAN via SM-CLP) plus the input-reliability
  limits that decide which console channel to use.
- skills/bmc-html5-kvm-addendum/README.md - proposed corrections to the
  installed bmc-html5-kvm skill, which documents iLO5 only.

## Remote

https://github.com/dmbyte/agent-skills

## How to contribute

```bash
git clone https://github.com/dmbyte/agent-skills.git
# add or correct a skill under skills/<name>/SKILL.md
git commit -am "..." && git push
```

Keep secrets out of this repository: no BMC passwords, no IP plans, no key
material, no password hashes. Use placeholders such as `<ilo4>` and refer to
where a credential lives instead of inlining it.
