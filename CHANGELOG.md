# Changelog

All notable changes to this project will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Unreleased

### Added
- New `/aap-first-time` skill for first-time local prerequisite setup — walks through ansible.cfg, secrets2, collections, SSH key pair, and vault file interactively (closes #3)

### Changed
- `/aap-bootstrap` now runs a preflight check before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #5)
- `/aap-setup-demo` now checks local prerequisites before starting — fails fast with a clear message and directs user to `/aap-first-time` if ansible.cfg, secrets2, or collections are missing (closes #7)
- `/aap-setup-demo` now checks if AAP is already bootstrapped before running bootstrap steps — skips straight to job launch if project and job template already exist (closes #7)

## [1.0.1] - 2026-04-23

### Fixed
- Replace `npx skillsadd` install command with native `claude plugins marketplace add` / `claude plugins install` commands (closes #1)

## [1.0.0] - 2026-04-23

Initial release — `aap-bootstrap` and `aap-setup-demo` skills.
