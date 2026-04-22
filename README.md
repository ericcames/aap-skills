# aap-skills

Claude Code skills for bootstrapping and demoing Ansible Automation Platform (AAP) instances provisioned from the Red Hat Demo Platform (RHDP).

## Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| aap-bootstrap | `/aap-bootstrap` | Bootstrap a fresh AAP instance — creates Hub credentials, Vault credential, project, and job template. Stops when AAP is ready. |
| aap-setup-demo | `/aap-setup-demo` | Bootstrap AAP **and** run `Setup - AAP - CAC` as a live demo story for a customer. |

## Install

```bash
npx skillsadd ericcames/aap-skills
```

## Prerequisites

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
  aap-bootstrap/
    SKILL.md
  aap-setup-demo/
    SKILL.md
```

## Related

- [aap.as.code](https://github.com/ericcames/aap.as.code) — bootstrap playbooks and CaC configuration
