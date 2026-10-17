# Passly Documentation

Passly is an open-source, team-oriented password manager built on an offline-first foundation. Unlike conventional web applications that rely on temporary caching, Passly maintains an autonomous, fully functional local replica on every client device. Users retain uninterrupted access to their credentials, TOTP codes, and passkeys at all times — even during indefinite server outages or complete network disconnection.

This directory contains the central architectural specifications, security designs, and task tracking records for the Passly ecosystem.

## Structure

```
docs/
├── architecture/    # Core technical specifications, security models, and sync protocols
├── deployment/      # Infrastructure deployment blueprints (OCI Free Tier, NixOS, Edge Proxy)
├── DONE/            # Documented records of completed milestones and verified features
├── TODO/            # Roadmap backlog, planned improvements, and pending initiatives
├── UA/              # Ukrainian localization of the architecture documentation
├── README.md        # Human-readable overview and directory map
└── AGENTS.md        # Guidelines and context for AI coding agents
```

For documentation in Ukrainian, see [docs/UA/README.md](UA/README.md).
