# aap-skills

Claude Code skills for bootstrapping and demoing Ansible Automation Platform (AAP) instances provisioned from the Red Hat Demo Platform (RHDP).

## First time here?

Run this first:

```
/aap-first-time
```

It walks you through every prerequisite interactively and validates each one before continuing. Takes about 10 minutes once.

Already set up? Skip to [Install](#install).

## Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| aap-first-time | `/aap-first-time` | First-time local setup — walks through every prerequisite interactively and validates each one. Run this once on a new machine before using the other skills. |
| aap-bootstrap | `/aap-bootstrap` | Bootstrap a fresh AAP instance — creates Hub credentials, Vault credential, project, and job template. Stops when AAP is ready. |
| aap-setup-demo | `/aap-setup-demo` | Bootstrap AAP **and** run `Setup - AAP - CAC` as a live demo story for a customer. |

## Workflow

**New user (first time on this machine):**
```
/aap-first-time → /aap-bootstrap → /aap-setup-demo
```

**Returning user (AAP already bootstrapped):**
```
/aap-setup-demo
```

**Just need AAP ready, no demo:**
```
/aap-bootstrap
```

## Install

```bash
claude plugins marketplace add ericcames/aap-skills
claude plugins install aap-skills
```

## Prerequisites

Run `/aap-first-time` to set these up interactively, or configure manually:

- Red Hat Demo Platform (RHDP) AAP instance provisioned
- `~/.ansible/ansible.cfg` with a valid Automation Hub API token under `[galaxy_server.rh_certified]`
- `~/.ansible/secrets2` containing your vault password (single line)
- Collections installed locally:

```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
  ansible-galaxy collection install ansible.platform ansible.controller \
  -p ./collections
```

## Repo Structure

```
skills/
  aap-first-time/
    SKILL.md
  aap-bootstrap/
    SKILL.md
  aap-setup-demo/
    SKILL.md
  references/
    aap-as-code-context.md
```

## Related

- [aap.as.code](https://github.com/ericcames/aap.as.code) — bootstrap playbooks and CaC configuration
