# Passly Workspace Notes

This repository is the umbrella project for Passly (`Svaroh/Passly`), binding
together the individual component repositories as Git submodules and housing the
central system documentation. Treat each component subdirectory as an independent
project and run component-specific git and build commands from the relevant
project root according to its local agent instructions.

The default branch `main` of `Svaroh/Passly` is protected: all changes to the
umbrella repository (including `docs/`, submodule pointer bumps, and root files)
must be introduced through feature branches and Pull Requests. Never push directly
to `main`.

## Communication & Security

- Maintain high security posture: never ask the user to paste private SSH keys,
  passwords, API keys, session tokens, SMTP credentials, or other secrets.
- Never store secrets or credentials anywhere in this workspace.
- Avoid absolute local filesystem paths in documentation and code; always use
  relative repository paths.

## Workspace Layout & Subproject Guidelines

Each component maintains its own detailed instructions, build steps, and testing
guidance. Refer to the corresponding local guide when working on that component:

- `passly_api/`
  - Backend API, authentication, and `sync/1` synchronization service.
  - Local instructions: `passly_api/AGENTS.md`.

- `passly_browser_extension/`
  - Client browser extension for Chromium, Firefox, and Safari.
  - Local instructions: `passly_browser_extension/AGENTS.md`.

- `passly_android/`
  - Native Android application with offline replica and background sync.
  - Local instructions: `passly_android/AGENTS.md`.

- `passly_styleguide/`
  - Shared UI design tokens, React components, LESS styles, and Storybook.
  - Local instructions: `passly_styleguide/AGENTS.md`.

- `docs/`
  - System architecture specifications, deployment guides, and task tracking.
  - Local instructions: `docs/AGENTS.md`.

## Passly Release Notes & Identifiers

- Do not add or invent `PB-` / Passbolt ticket prefixes in new Passly release
  notes, changelog entries, GitHub releases, commit summaries, or deployment
  summaries. Use plain descriptions, or real Passly issue IDs only when they
  actually exist. Existing historical upstream Passbolt changelog entries may
  keep their original identifiers.
- Prefer neutral internal names for new query parameters, storage keys, helper
  names, and identifiers. Avoid `Passly` / `passbolt` prefixes unless the name
  is user-visible branding, required compatibility, or an existing protocol.

## Editing Rules

- Preserve unrelated user changes: check `git status --short` inside the
  relevant project directory before making edits.
- Do not run destructive git commands unless explicitly requested.
- Prefer project-local patterns, standards, and tools documented in each
  subproject's guide.
- For new files that are fully Passly/Svaroh-authored, use the Passly/Svaroh
  copyright header, including `@link https://passly.svaroh.net Passly`. Keep
  Passbolt headers only in files that are derived from or modified from upstream
  Passbolt code.
- Always run focused tests locally in the changed component before committing,
  and report if any test could not be run.
