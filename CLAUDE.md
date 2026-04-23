# aap-skills — Claude Code Guidance

This repo is a Claude Code plugin marketplace. It ships three skills that bootstrap and demo Ansible Automation Platform (AAP) instances provisioned from Red Hat Demo Platform (RHDP).

## Runtime Context — Important

**The skills in this repo run from inside the [`aap.as.code`](https://github.com/ericcames/aap.as.code) repo, not from this one.**

This repo only *defines* the skills. When a user invokes `/aap-bootstrap` or `/aap-setup-demo`, Claude Code is expected to be running from a clone of `aap.as.code` — because the skills execute `playbooks/bootstrap_dev.yml`, install collections into `./collections/`, and write inventories under `inventories/`. `/aap-first-time` and `/aap-bootstrap` both verify this and stop with a clone instruction if not.

When editing skills, keep that split in mind:
- Paths inside the skills (`./collections/...`, `playbooks/...`, `inventories/...`) are relative to `aap.as.code`, not this repo.
- User-facing state lives in `~/.ansible/ansible.cfg` and `~/.ansible/secrets2`.

## Repo Layout

```
skills/
  aap-first-time/SKILL.md          prerequisite setup (run once per machine)
  aap-bootstrap/SKILL.md           credentials + inventory + bootstrap playbook
  aap-setup-demo/SKILL.md          bootstrap + launch Setup - AAP - CAC as a demo
  references/
    aap-as-code-context.md         shared reference — object names, API paths, vault vars
.claude/skills/<name>              symlinks to skills/<name> — enables project-level
                                   auto-discovery when Claude Code is opened in a
                                   fresh clone, without the marketplace install
.claude-plugin/marketplace.json    marketplace manifest; skills[] drives
                                   the `claude plugins install` path
```

`skills/<name>/SKILL.md` is the single source of truth. `.claude/skills/<name>` is a symlink pointing at it — always edit the file under `skills/`, never the link.

The three skills form a pipeline: `/aap-first-time` → `/aap-bootstrap` → `/aap-setup-demo`. Each later skill assumes the earlier ones ran, but also preflights and redirects if prerequisites are missing.

## Skill Conventions

Every skill is a single `SKILL.md` with YAML frontmatter:

```yaml
---
name: <slug matching directory name>
description: "<when to trigger; TRIGGER when: ...; SKIP: ...>"
---
```

The `description` is what Claude Code uses to decide when a skill is relevant. Keep the `TRIGGER when:` / `SKIP:` clauses explicit — they are load-bearing for activation, not documentation. Existing skills are the reference template; match their voice.

## Shared Reference Pattern

AAP object names, API endpoints, vault URL patterns, and the full vault-variable table live in `skills/references/aap-as-code-context.md`. **Skills should cite this file rather than duplicate its content.** If a new fact applies to more than one skill, add it to the reference file and link from the skills.

## CHANGELOG Discipline

Every user-visible change gets an entry in `CHANGELOG.md` under `## Unreleased`, following [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) categories (`Added` / `Changed` / `Fixed` / `Removed`). Existing entries reference the GitHub issue they close (`(closes #N)`) — follow that style. Releases move `Unreleased` content under a new `## [x.y.z] - YYYY-MM-DD` heading.

## Verifying Changes

There are no automated tests. Two loops, shortest first:

1. **Does the skill load?** — `claude .` in a fresh clone of this repo. The `.claude/skills/` symlinks make all three skills visible as slash commands immediately. Good for catching frontmatter typos, broken `description` fields, or renames that miss the symlink.
2. **Does the skill run?** — install the plugin (`claude plugins install <path>` locally, or `claude plugins marketplace add ericcames/aap-skills` after pushing) and invoke the slash command from a clone of `aap.as.code`. Walk the changed branch(es) end-to-end — preflight, credential resolution, playbook invocation, API verification, final summary.

Content-only edits (README, CHANGELOG, references) can be reviewed by reading the diff.

## Related

- [aap.as.code](https://github.com/ericcames/aap.as.code) — the playbooks and CaC content the skills drive
- [sourcefiles](https://github.com/ericcames/sourcefiles) — example public host for the vault file and SSH public key
