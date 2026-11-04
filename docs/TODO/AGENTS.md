# Passly: Upcoming Tasks & Backlog

This directory (`docs/TODO/`) tracks pending architectural requirements, upcoming development tasks, and roadmap items for the Passly ecosystem.

## Purpose & Scope

- Outlines planned features, technical debt reduction, and architectural improvements.
- Tracks pending system requirements (such as R7 Offline Writes and R8 Emergency Backup Export).
- Organizes implementation specs and task checklists prior to active development.
- Active backlog document: [roadmap-backlog.md](roadmap-backlog.md).

## Guidelines for Agents

- **Defining Tasks**: Structure items with clear objectives, affected subprojects (`passly_api`, `passly_browser_extension`, `passly_android`, `passly_styleguide`), and verifiable acceptance criteria.
- **Architectural Alignment**: Ensure all planned designs adhere to Passly's core requirement (R0: full offline autonomy) and cryptographic guidelines.
- **Task Migration (No `[COMPLETED]` tags)**: Never mark completed tasks or milestones with `[COMPLETED]` or `[DONE]` inline inside `docs/TODO/`. Once an initiative is completed and verified, completely remove it from `docs/TODO/` and transfer its documentation and verification records into `docs/DONE/`, fully updating `docs/DONE/implementation-status.md` and all related documentation.
- **Naming**: Use clear, descriptive titles without invented `PB-` prefixes.
