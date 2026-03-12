# Contributing to GNAP

GNAP is a protocol specification — contributions are welcome.

## What to contribute

- **Protocol improvements** — propose changes to the four entities or transport rules
- **Clarifications** — if something in the spec is ambiguous, open an issue
- **Examples** — add example configurations for different team setups
- **Integrations** — build tools on top of GNAP (CLI, GitHub Actions, bridges)
- **Translations** — translate the spec into other languages

## How

1. Open an issue describing your proposal
2. Fork the repo
3. Make your changes
4. Submit a PR

## Guidelines

- GNAP is intentionally minimal. Resist adding entities to the protocol.
- Operational concerns (budget enforcement, concurrency control, stall detection) belong in the application layer, not the protocol.
- Keep examples valid — they should always match the schemas in README.md.
- The spec is the single source of truth — README.md defines the protocol.

## Code of Conduct

Be respectful. Be constructive. Focus on the work.
