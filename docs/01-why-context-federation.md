# Why Context Federation

> **⚠️ DRAFT.** Working draft; subject to change.

## The problem

An AI agent is only as good as the context it can see. In practice, the context an
agent needs is scattered across many systems: a CRM holds companies, contacts, and
deals; an accounting system holds ledgers and invoices; a calendar holds events; a
support tool holds tickets. Each of these is a *system of record* for some slice of
the user's world.

Agents typically reach this data in one of two unsatisfying ways:

1. **Bespoke integrations** — a custom connector per system, each with its own auth,
   shape, and refresh logic. These are expensive to build and brittle to maintain.
2. **Tool calls at query time** — the agent calls an API live whenever it needs
   something. This works for actions, but it makes *context* (the background the
   agent reasons over) slow, rate‑limited, and invisible to local reasoning,
   ranking, and offline use.

What's missing is a **uniform way to bring context in** — to mirror the relevant
parts of those systems into the agent's own context store, keep them current, know
where each piece came from, and write changes back — without inventing a new
integration each time.

## What CF/x3 is

CF/x3 (Context Federation) is a small protocol for exactly that. A **client**
connects to one or more **sources** and:

- **discovers** what context a source offers — its *context types*, organized as a
  **tree** rooted in tier‑1 *base categories* (the "life primitives") — and what it
  may do with them (read / write / delete) — the **manifest**;
- **reads** context in bulk, either a full snapshot or only what changed since the
  last sync — **sync**;
- **mirrors** those records into its own store, **tagged with the originating
  source**, idempotently (re‑syncing never duplicates);
- **writes** context back — create / update / delete nodes (and, where the source
  allows, types) — **write**.

Because it is uniform, one client implementation talks to every CF/x3 source, and
one server implementation exposes a system to every CF/x3 client.

## Scope: context only

CF/x3 handles **context** — descriptive state the agent reasons over (a company, a
person, an account, a note). It is intentionally **not** a tool/action protocol.

- **In scope:** CRUD on *context types* and *context nodes*.
- **Out of scope:** domain actions with side effects beyond the context record
  itself — "win this deal", "charge this card", "send this email". Those belong on a
  separate surface, e.g. an **MCP** server.

This separation is a feature. A CF/x3 client can sync and mirror a source without
any risk of triggering domain side effects, and a server author reasons about
federation and actions independently.

## How it relates to other protocols

- **A2A (Agent‑to‑Agent):** CF/x3 is a *profile* of A2A. It reuses A2A's agent‑card
  discovery and JSON‑RPC `tasks/send` transport, and defines three well‑known
  *skills* plus the JSON shapes they carry. If you already speak A2A, CF/x3 is a
  thin layer on top.
- **MCP (Model Context Protocol):** complementary. Use **CF/x3 for context**
  (federated read/write of records) and **MCP for tools** (domain actions). A single
  system commonly exposes both: a CF/x3 server for its context and an MCP server for
  its actions.
- **Bulk export formats (e.g. JSON‑LD dumps):** earlier federation attempts shipped
  whole‑graph exports. CF/x3 keeps the useful parts — stable IDs, tombstones, an
  incremental cursor — but as a compact JSON request/response over A2A, and adds the
  missing piece those approaches lacked: **a defined write path and idempotent
  ingest keyed on origin**.

## Design goals

1. **Context‑only and predictable.** Every call reads or writes context. No hidden
   side effects.
2. **Source attribution.** Every mirrored node records which source it came from and
   its remote id, so the consumer always knows provenance and can re‑sync or detach a
   source cleanly.
3. **Idempotent.** A stable `uid` per node + upsert keyed on `(source, uid)` means
   repeated syncs converge; they never duplicate.
4. **Incremental by default.** The client carries a cursor (`since`) and the server
   returns the next one (`syncedAt`); a full sync is just `since: null`.
5. **Permissioned.** The manifest is filtered to what the caller may actually see and
   do; writes are gated server‑side.
6. **Implementable from the spec.** The wire format is small enough that an agent can
   build a conformant server or client from these documents.

## Roles

- **Source / server:** a system that exposes some of its data as CF/x3 context types
  and nodes (e.g. a CRM exposing `company`, `contact`, `deal`).
- **Consumer / client:** an agent or app that connects to sources, mirrors their
  context locally, and optionally writes back.

Read next: the [Protocol specification](02-protocol.md).
