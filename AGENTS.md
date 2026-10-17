# Passly Workspace Notes

This directory is the umbrella repository for Passly (`Svaroh/Passly`), binding
together the individual component repositories as Git submodules and housing the
central system documentation. Treat each component subdirectory as its own project
and run component-specific git and build commands from the relevant project root.

The default branch `main` of `Svaroh/Passly` is protected: all changes must be
introduced through Pull Requests.

## Communication

- The user cares about security posture. Do not ask them to paste private SSH
  keys, passwords, API keys, session tokens, SMTP credentials, or other secrets.
- Do not store secrets in this workspace.

## Workspace Layout

- `passly_api/`
  - Git repo currently on `main`.
  - Remotes observed: `origin` is `Svaroh/passly_api`,
    `legacy-66ton99` is `66Ton99/passbolt_api`, and `upstream` is
    `passbolt/passbolt_api`.
  - GitHub default branch: `main`.
  - Stack: CakePHP/PHP application with Composer, PHPUnit, PHPStan, Psalm,
    phpcs, npm lockfile linting, and Grunt/styleguide assets.
  - Local Nix dev shell: `flake.nix` / `flake.lock` provide the preferred
    local toolchain for API work and tests. Use `nix develop` from
    `passly_api/` before running Composer, PHPUnit, npm, GPG, or MySQL client
    commands. Keep this Nix environment updated when PHP extensions,
    Composer/npm tooling, test CLI tools, or local test requirements change.
  - Important local instructions: `passly_api/.codex/AGENTS.md`.
  - Main areas: `src/Controller`, `src/Model`, `src/Service`,
    `plugins/PassboltCe/*`, `tests/TestCase`, `resources/locales`.

- `passly_browser_extension/`
  - Git repo currently on `main`.
  - Remotes observed: `origin` is `Svaroh/passly_browser_extension`,
    `legacy-66ton99` is `66Ton99/passbolt_browser_extension`.
  - GitHub default branch: `main`.
  - Stack: JavaScript/React browser extension with Grunt, Webpack, Jest, ESLint.
  - Important local instructions: `passly_browser_extension/.codex/AGENTS.md`.
  - Main areas: `src/all`, `src/chrome`, `src/chrome-mv3`, `src/firefox`,
    `src/safari`, `build-safari-extension`, `test`.
  - On `2026-06-02`, the `/app/passwords` top-left workspace logo and
    QuickAccess popup logo were fixed in extension commits `38a69c73` and
    `002e0bf2`; they are bundled from `passbolt-styleguide` into the browser
    extension `webAccessibleResources/js/dist/*.js`, not served by the API
    `/img/logo` files.
  - On `2026-06-02`, browser first-login import was fixed in commit
    `845b5265` after production showed `/auth/verify.json?api-version=v2`
    returning "You are already logged in" when an existing Passbolt session
    cookie was included. The extension now verifies the transferred key with
    `credentials: "omit"` for browser first-login only, so the server performs
    the GPGAuth verify challenge without the current browser session. Current
    rebuilt extension artifact version: `5.12.87`.
  - On `2026-06-02`, browser first-login UI translations were completed for
    hardcoded controller errors in `MobileTransferEntrypoint.js`; the
    Ukrainian bundle now translates the private-key/server mismatch and related
    first-login validation errors. Current rebuilt extension artifact version:
    `5.12.88`.

- `passly_android/`
  - Git repo currently on `main`.
  - Remotes observed: `origin` is `Svaroh/passly_android`,
    `legacy-66ton99` is `66Ton99/mobile-passbolt-android`.
  - GitHub default branch: `main`.
  - Stack: Kotlin/Android Gradle multi-module project.
  - Important local instructions: `passly_android/.codex/AGENTS.md`.
  - Main module groups: `app`, `core/*`, `features/*`, `domain/*`,
    `services/*`.
  - For routine Android app changes, bump the visible `projectVersionName`;
    do not bump the internal `projectVersionCode` unless explicitly requested
    or preparing a store/release build that requires a higher internal code.

- `passly_styleguide/`
  - Git repo currently on `passly/rebrand` (targeting `main`).
  - Remote: `origin` is `Svaroh/passly_styleguide`.
  - GitHub default branch: `main`.
  - Stack: React / LESS / Storybook / Webpack styleguide package with Jest,
    ESLint, and Playwright test-runner.
  - Main areas: `src/react-extension`, `src/react-quickaccess`, `src/less`,
    `src/locales`, `.storybook`.

- `docs/`
  - Central architectural specifications, cryptographic designs, deployment guides, and task tracking.
  - `architecture/`: Normative technical docs (system overview, sync protocol `sync/1`,
    security & encryption, client offline replicas).
  - `deployment/`: Production deployment blueprints (OCI Free Tier, Edge DMZ reverse proxy, NixOS operational guides).
  - `UA/`: Complete Ukrainian mirror of the architecture documentation.
  - `DONE/`: Archive and summaries of completed features and landed milestones.
  - `TODO/`: Roadmap backlog and pending requirements (R7 offline writes, R8 emergency export).
  - Local agent instructions: `docs/AGENTS.md`.

## Branch / Repository Naming

- On `2026-06-02`, local project directories were renamed to match the
  `Svaroh` GitHub repository names:
  - `passbolt_api` -> `passly_api`
  - `passbolt_browser_extension` -> `passly_browser_extension`
  - `mobile-passbolt-android` -> `passly_android`
- On `2026-06-02`, the current Passly rebrand branches were promoted to
  `main` locally and remotely, and GitHub default branches were switched to
  `main` for:
  - `Svaroh/passly_api`
  - `Svaroh/passly_browser_extension`
  - `Svaroh/passly_android`
- Current `main` heads after the migration:
  - API: `06f3a49116` (`Replace inline API logos with Passly Team Shield`)
  - Browser extension: `f6243e9d` (`Add QuickAccess password generator`)
  - Android: `0d5ecbb3f` (`Replace Android branding with Passly Team Shield`)
- Old `master` and `rebrand/passly` refs may still exist locally or on GitHub
  for history/backward compatibility. Use `main` for new work unless the user
  explicitly asks for a legacy branch.
- On `2026-06-02`, the newest Passly commits were rewritten once more to fix
  author/committer metadata from
  `45160296+66Ton99@users.noreply.github.com` to `github@66ton99.org.ua`.
  The rewritten `main` heads were force-pushed to both `origin/main` and the
  legacy `origin/rebrand/passly` refs.

## Passly Release Notes

- Do not add or invent `PB-` / Passbolt ticket prefixes in new Passly release
  notes, changelog entries, GitHub releases, commit summaries, or deployment
  summaries. Use plain descriptions, or real Passly issue IDs only when they
  actually exist. Existing historical upstream Passbolt changelog entries may
  keep their original identifiers.

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

Android app versioning and device-install rules are documented in
`passly_android/.codex/AGENTS.md`.

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

## API Production Safety

The API checkout has extra operational notes in `passly_api/.codex/AGENTS.md`.
Read that file before doing API deployment, OCI, NixOS, or remote server work.

Important rule: never run automated tests on the remote Passbolt production
server. This includes PHPUnit, Composer test scripts, fixtures, Selenium, test
database setup, or any command that enables test mode. Run tests only locally in
the appropriate local checkout.

Remote verification should stay production-safe: service status, logs, Nginx /
PHP-FPM status, health checks, HTTP(S) endpoint checks, and read-only
configuration inspection.

Do not create ad-hoc backup directories or backup copies on production servers
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
- Prefer neutral internal names for new query parameters, storage keys, helper
  names, and identifiers. Avoid `Passly` / `passbolt` prefixes unless the name
  is user-visible branding, required compatibility, or an existing protocol.
- Use focused tests for the changed project and report if a test could not be
  run.
- For changes to the root umbrella repository (`Svaroh/Passly`, including `docs/`,
  submodule references, and root configuration files), never push directly to `main`.
  Create a branch and open a Pull Request.
- Do not silence PHPUnit coverage metadata warnings with flags such as
  `--do-not-fail-on-phpunit-warning`; fix the invalid `@covers` / `@uses`
  annotations or the underlying metadata instead.
