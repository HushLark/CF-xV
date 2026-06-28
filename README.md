<p align="center">
  <img src="assets/cfx3-logo.png" alt="CF/x3" width="420">
</p>

# CF/x3 — Context Federation Protocol

> **⚠️ DRAFT.** This specification is a working draft and may change. Do not treat
> it as stable. Versioned releases will be tagged when it stabilizes.

CF/x3 is a small, open protocol for **federating context** between systems: a
client pulls **context types** and **context nodes** from one or more remote
sources into its own store — tagged with their origin — and can create, update,
and delete that context back at the source. It is a thin **profile of A2A**
(Agent‑to‑Agent), so it inherits A2A's discovery and transport and adds a compact
JSON interface focused entirely on context.

CF/x3 deliberately does **only context** (CRUD on context types and nodes). It is
*not* a tool/action protocol — domain actions (e.g. "mark a deal won", "send an
email") belong to a separate surface such as an **MCP** server. Keeping CF/x3
context‑only makes federation predictable: every CF/x3 call reads or writes
context, nothing else.

This repository is the documentation site for the protocol. It is written so an
**AI agent (or a human) can implement either side — server, client, or both —
from these docs alone.**

## Documents

| Doc | What it covers |
|-----|----------------|
| [Why context federation](docs/01-why-context-federation.md) | The problem CF/x3 solves and how it relates to A2A, MCP, and prior approaches |
| [Protocol specification](docs/02-protocol.md) | The wire format: transport, skills, data model, semantics, errors, examples |
| [Building a server](docs/03-server.md) | How to expose your system as a CF/x3 source |
| [Building a client](docs/04-client.md) | How to consume CF/x3 sources and mirror context into your store |

## At a glance

- **Transport:** A2A — `GET /.well-known/agent.json` for discovery, JSON‑RPC 2.0
  `tasks/send` to a single POST endpoint, results returned as task *artifacts*.
- **Auth:** OAuth 2.0 **Bearer** token (RFC 6750) on every request. Every skill —
  including the manifest — requires a valid token.
- **Permissions:** a **graded per‑type level** — `0 none · 1 read · 2 update ·
  3 create · 4 delete`, each building on the previous. A caller may have a different
  level on each context type; the server maps its RBAC to levels.
- **Sessionless:** no login/handshake/session — every request is authenticated and
  authorized from its bearer token, and the sync cursor is client‑owned.
- **Hierarchical context types:** types form a **tree**, not a flat list. The manifest
  declares tier‑1 **base categories** (the "life primitives" roots — `person`, `place`,
  `thing`, `event`, `task`, `goal`), and each type places itself via `baseCategory`
  (its root) and optional `parent` (a more‑specific subtype). Depth = specificity.
- **Four skills:**
  - `cfx3.manifest` — the context‑type **tree** (base categories + types the caller may read) + the caller's level on each.
  - `cfx3.permissions` — the **calling principal's** subject + per‑type level map
    (the "who am I / what can I do" call).
  - `cfx3.sync` — **read** context; full (`since: null`) or incremental (`since: <ISO>`).
  - `cfx3.write` — **create / update / delete** context nodes (and, where allowed, types).
- **Idempotent mirroring:** every node has a stable `uid`; clients upsert keyed on
  `(source, uid)`, so re‑syncs never duplicate.
- **Client‑owned cursor:** the server returns `syncedAt`; the client stores it and
  sends it back as the next `since`.

## Status & versioning

Current protocol version: **`cfx3_version: 1`** (DRAFT). Breaking changes will
bump this integer. Servers advertise it in the manifest; clients should check it.

## License

See [LICENSE](LICENSE).
