# Passly

Passly is an open-source, team-oriented password manager designed around complete local autonomy and end-to-end encryption. Unlike traditional password managers that depend on continuous server connectivity, Passly ensures your encrypted credentials, TOTP codes, and passkeys are always fully functional on your device — even during indefinite server downtime or offline use.

## Ecosystem

This umbrella repository organizes the core projects of the Passly platform:

- **[passly_api](passly_api)** — Backend API service and delta-synchronization engine.
- **[passly_browser_extension](passly_browser_extension)** — Web extension for Chromium, Firefox, and Safari featuring an autonomous local workspace.
- **[passly_android](passly_android)** — Native Android application backed by Room and SQLCipher.
- **[passly_styleguide](passly_styleguide)** — Shared UI component library, design tokens, and styleguide.
- **[docs](docs)** — Architectural specifications, cryptographic models, and roadmap.

## Getting Started

Clone the repository with all submodules:

```bash
git clone --recurse-submodules git@github.com:Svaroh/Passly.git
cd Passly
```

If already cloned, initialize and update submodules:

```bash
git submodule update --init --recursive
```

## Documentation

- Architecture and technical specifications: [docs/README.md](docs/README.md)
- Українська документація: [docs/UA/README.md](docs/UA/README.md)
