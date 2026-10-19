# Passly Workspace Notes

This directory is the umbrella repository for Passly (`Svaroh/Passly`), binding
together the individual component repositories as Git submodules and housing the
central system documentation. Treat each component subdirectory as its own project
and run component-specific git and build commands from the relevant project root.

The default branch `main` of `Svaroh/Passly` is protected: all changes must be
introduced through Pull Requests.

## Communication & Security

- The user cares about security posture. Do not ask them to paste private SSH
  keys, passwords, API keys, session tokens, SMTP credentials, or other secrets.
- Do not store secrets in this workspace.
- Avoid absolute local filesystem paths in documentation and code; use relative
  repository paths.

## Workspace Layout

- `passly_api/`
  - Stack: CakePHP / PHP backend with Composer, PHPUnit, PHPStan, Psalm, phpcs,
    npm lockfile linting, and Grunt assets.
  - Local Nix dev shell: `flake.nix` / `flake.lock` provide the preferred local
    toolchain for API work and tests. Run `nix develop` from `passly_api/`
    before running Composer, PHPUnit, npm, GPG, or MySQL client commands.
  - Main areas: `src/Controller`, `src/Model`, `src/Service`,
    `plugins/PassboltCe/*` (including `Sync` and `OfflineMode`),
    `tests/TestCase`, `resources/locales`.
  - Local instructions: `passly_api/.codex/AGENTS.md`.

- `passly_browser_extension/`
  - Stack: JavaScript / React browser extension (Chromium, Firefox, Safari) with
    Grunt, Webpack, Jest, ESLint, and Playwright.
  - Main areas: `src/all`, `src/chrome`, `src/chrome-mv3`, `src/firefox`,
    `src/safari`, `build-safari-extension`, `test`.
  - Key capabilities: Containerized local workspace (`workspace.html`),
    offline-first session unlock, local secret persistence, WebCrypto, and
    biometric unlock.
  - Local instructions: `passly_browser_extension/.codex/AGENTS.md`.

- `passly_android/`
  - Stack: Kotlin / Android Gradle multi-module project.
  - Main module groups: `app`, `core/*` (including `core/sync`), `features/*`,
    `domain/*`, `services/*`.
  - Key capabilities: Room + SQLCipher local replica, hardware Keystore,
    non-blocking offline unlock, and background synchronization.
  - For routine Android app changes, bump the visible `projectVersionName`;
    do not bump the internal `projectVersionCode` unless explicitly requested
    or preparing a store/release build that requires a higher internal code.
  - Local instructions: `passly_android/.codex/AGENTS.md`.

- `passly_styleguide/`
  - Stack: React / LESS / Storybook / Webpack styleguide package with Jest,
    ESLint, and Playwright test-runner.
  - Main areas: `src/react-extension`, `src/react-quickaccess`, `src/less`,
    `src/locales`, `.storybook`.
  - Key capabilities: Shared UI design tokens, offline workspace states, and
    biometric dialogs.

- `docs/`
  - Central architectural specifications, cryptographic designs, deployment
    guides, and task tracking.
  - `architecture/`: Normative technical specifications (system overview, sync
    protocol `sync/1`, security & encryption, client offline replicas).
  - `deployment/`: Production deployment blueprints (OCI Free Tier, Edge DMZ
    reverse proxy, NixOS operational guides).
  - `DONE/`: Landed milestones and verified capability audits.
  - `TODO/`: Roadmap backlog and pending requirements (such as offline writes
    and emergency export).
  - `UA/`: Complete Ukrainian mirror of the documentation.
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

## Common Commands

Use these from the relevant project root.

### API

Prefer the repo Nix shell first:

```bash
nix develop
```

Update `flake.nix` / `flake.lock` as needed when local API tooling or test
dependencies change.

```bash
composer test
composer cs-check
composer cs-fix
composer stan
composer psalm
npm run lint
```

### Browser Extension

```bash
npm ci
npm run build
npm run lint
npm run test:unit
grunt build
```

For routine extension work, prefer the narrower commands documented in
`passly_browser_extension/.codex/AGENTS.md`.

### Android

```bash
./gradlew assembleDebug
./gradlew detekt ktlint lintDebug unitTest licenseeRelease dependencyUpdates buildHealth
./gradlew connectedAndroidTest
```

### Styleguide

```bash
npm ci
npm run lint
npm run build
npm run test:unit
npm run test:ci:coverage
npm run dev:storybook:build
npm run test:storybook
```

## Production Safety & Remote Servers

The API checkout has operational notes in `passly_api/.codex/AGENTS.md`.
Read that file before doing API deployment, OCI, NixOS, or remote server work.

- **Never run automated tests on remote production servers.** This includes
  PHPUnit, Composer test scripts, fixtures, Selenium, test database setup, or
  any command that enables test mode. Run tests only locally in the appropriate
  local checkout.
- Remote verification must stay production-safe and read-only: service status,
  journalctl logs, Nginx / PHP-FPM status, health checks, HTTP(S) endpoint
  checks, and read-only configuration inspection.
- Do not create ad-hoc backup directories or backup copies on production servers
  during deploys unless the user explicitly asks for one. Use git history and
  build artifacts as the rollback source.

## Editing Rules

- Preserve unrelated user changes. Check `git status --short` inside the
  relevant subproject before broad edits.
- Do not run destructive git commands unless explicitly requested.
- Prefer project-local patterns and tools.
- For new files that are fully Passly/Svaroh-authored, use the Passly/Svaroh
  copyright header, including `@link https://passly.svaroh.net Passly`. Do not
  copy the Passbolt SA copyright header into new Passly-owned files; keep
  Passbolt headers only in files that are derived from or modified from
  upstream Passbolt code.
- Use focused tests for the changed project and report if a test could not be
  run.
- For changes to the root umbrella repository (`Svaroh/Passly`, including `docs/`,
  submodule references, and root configuration files), never push directly to `main`.
  Create a branch and open a Pull Request.
- Do not silence PHPUnit coverage metadata warnings with flags such as
  `--do-not-fail-on-phpunit-warning`; fix the invalid `@covers` / `@uses`
  annotations or the underlying metadata instead.
