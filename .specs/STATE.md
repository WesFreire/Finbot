# STATE

## Decisions

### AD-001
- **Decision**: The MVP receives Telegram updates by grammY long polling inside the Nest process and does not open an HTTP server.
- **Reason**: The conversational spec does not require a public URL or Swagger, and grammY `bot.start()` handles updates one at a time, which matches one pending draft per user.
- **Trade-off**: The process must stay up to receive messages. A later webhook or HTTP feature has to supersede this decision.
- **Scope**: Every feature until a spec adds an HTTP surface.
- **Date**: 2026-11-10
- **Status**: active

### AD-002
- **Decision**: Database integration tests run against PostgreSQL started by `@testcontainers/postgresql`, not an in-memory stand-in.
- **Reason**: Case-insensitive unique names, one pending draft per user, and integer centavos have to be real PostgreSQL behavior.
- **Trade-off**: The full gate and the build gate need a container runtime.
- **Scope**: Every feature that persists to PostgreSQL.
- **Date**: 2026-11-10
- **Status**: active

## Handoff
