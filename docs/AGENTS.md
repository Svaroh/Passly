# Documentation Agent Guidelines

This directory (`docs/`) contains the architectural blueprints, technical specifications, and task tracking for the Passly password management platform.

## Directory Layout

- `architecture/`: Normative technical specifications (system overview, cryptographic models, `sync/1` protocol, and platform-specific replica engines).
- `deployment/`: Production deployment guides, infrastructure blueprints (OCI Free Tier, edge reverse proxies), and NixOS operational guidelines.
- `DONE/`: Summaries of implemented features, closed milestones, audit records, and verification reports.
- `TODO/`: Roadmap backlog, planned enhancements, and pending requirements (such as offline writes and emergency export).
- `UA/`: Complete Ukrainian localization of the architecture specifications.

## Operating Principles for Agents

- **Sync with Code**: Ensure architectural documents accurately reflect current implementations in `passly_api`, `passly_browser_extension`, `passly_android`, and `passly_styleguide`.
- **Localization Parity**: When updating foundational concepts in `architecture/`, maintain parity with `UA/architecture/`.
- **Naming & Identifiers**: Do not invent `PB-` or Passbolt ticket prefixes. Use plain descriptions or verified Passly issue IDs.
- **Security & Hygiene**: Never insert private keys, tokens, session data, or production credentials into documentation.
