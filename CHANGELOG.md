# Changelog

All notable changes to this project will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Unreleased

### Added
- New `/aap-first-time` skill for first-time local prerequisite setup
- `skills/references/aap-as-code-context.md` — shared reference file with AAP object names, API endpoints, vault URL patterns, and full vault variable table (closes #15) — walks through ansible.cfg, secrets2, collections, SSH key pair, and vault file interactively (closes #3)

### Changed
- `/aap-bootstrap` Step 5 now includes idempotency guidance — `ok` results mean an object already exists and is correct, not a warning; re-running after a partial failure is safe (closes #11)
- `/aap-bootstrap` now runs a preflight check before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #5)
- `/aap-bootstrap` Step 6 now verifies each created AAP object via API and prints a structured success summary; vault credential verified by type so any name is supported (closes #9)
- `/aap-setup-demo` now asks for audience (Technical / Management / Mixed) before launching and provides a tailored narration track for each (closes #13)
- `/aap-setup-demo` now checks local prerequisites before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #7)
- `/aap-setup-demo` now checks if AAP is already bootstrapped before running bootstrap steps — skips straight to job launch if project and job template already exist (closes #7)

## [1.0.1] - 2026-04-23

### Fixed
- Replace `npx skillsadd` install command with native `claude plugins marketplace add` / `claude plugins install` commands (closes #1)

## [1.0.0] - 2026-04-23

Initial release — `aap-bootstrap` and `aap-setup-demo` skills.
