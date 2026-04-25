# Changelog

All notable changes to this project will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Unreleased

### Added
- New `/aap-first-time` skill walks through ansible.cfg, secrets2, collections, SSH key pair, and vault file interactively (closes #3)
- New shared reference file `skills/references/aap-as-code-context.md` with AAP object names, API endpoints, vault URL patterns, and full vault variable table (closes #15)

### Changed
- `/aap-first-time` Step 2 now writes `~/.ansible/ansible.cfg` directly after the user provides their Hub token — no manual file creation required; template updated to include `[galaxy]` and `[galaxy_server.community]` sections (closes #24)
- `/aap-first-time` now collects `my_vault` and `my_windows_catalog_short_description` and writes all four user-specific vars to `inventories/rhdp-sample-demo/group_vars/all.yml` (closes #25)
- `/aap-bootstrap` and `/aap-setup-demo` now check `inventories/rhdp-sample-demo/group_vars/all.yml` at startup and halt if `my_vault`, `my_remote_vault`, or `my_remote_ssh_pub_key` are empty (closes #25)
- `/aap-bootstrap` Step 4 now reads user-specific vars from `rhdp-sample-demo/group_vars/all.yml` instead of hardcoding them (closes #25)
- `/aap-bootstrap` Step 5 now includes idempotency guidance — `ok` results mean an object already exists and is correct, not a warning; re-running after a partial failure is safe (closes #11)
- `/aap-bootstrap` now runs a preflight check before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #5)
- `/aap-bootstrap` Step 6 now verifies each created AAP object via API and prints a structured success summary; vault credential verified by type so any name is supported (closes #9)
- `/aap-setup-demo` audience selection and narration tracks removed (closes #27)
- `/aap-setup-demo` now checks local prerequisites before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #7)
- `/aap-setup-demo` now checks if AAP is already bootstrapped before running bootstrap steps — skips straight to job launch if project and job template already exist (closes #7)
- README: added "First time here?" section pointing to `/aap-first-time`, workflow diagram showing three user paths, updated repo structure (closes #17)

### Fixed
- `/aap-first-time` now checks that the user is running from inside the `aap.as.code` repo before proceeding — stops with a clear clone instruction if not (closes #19)
- README "First time here?" section now includes cloning `aap.as.code` as step 0 (closes #19)

## [1.0.1] - 2026-04-23

### Fixed
- Replace `npx skillsadd` install command with native `claude plugins marketplace add` / `claude plugins install` commands (closes #1)

## [1.0.0] - 2026-04-23

Initial release — `aap-bootstrap` and `aap-setup-demo` skills.
