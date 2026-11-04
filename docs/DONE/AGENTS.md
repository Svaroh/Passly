# Passly: Completed Work & Landed Features

This directory (`docs/DONE/`) tracks delivered milestones, implemented architectural requirements, and verified features across the Passly ecosystem.

## Purpose & Scope

- Preserves records and verification notes for completed tasks.
- Documents shipped capabilities across `passly_api`, `passly_browser_extension`, `passly_android`, and `passly_styleguide`.
- Serves as the historical source of truth for initiatives transitioned out of `docs/TODO/`.
- Verified milestones audit: [implementation-status.md](implementation-status.md).

## Guidelines for Agents

- **Moving Items from TODO (No `[COMPLETED]` tags in TODO)**: Never leave completed items marked as `[COMPLETED]` in `docs/TODO/`. When a task meets its Definition of Done and passes verification, completely remove it from `docs/TODO/` and move its documentation and verification reports here, updating `docs/DONE/implementation-status.md` and related references.
- **Traceability**: Reference affected subprojects, key commits, or PRs where applicable.
- **Conventions**: Keep records concise, factual, and free of fictitious `PB-` ticket prefixes.
- **Security**: Never log real credentials, cryptographic keys, or private tokens in completion reports.
