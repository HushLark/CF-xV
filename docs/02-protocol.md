# CF/x3 — Protocol Specification

> **⚠️ DRAFT.** Working draft (`cfx3_version: 1`). The wire format may change before
> a stable release. Implementations should check `cfx3_version` (see [Versioning](#versioning)).

CF/x3 is a profile of **A2A**. This document is the normative wire spec: an
implementer can build a conformant **server** or **client** from it alone. Key
words **MUST**, **SHOULD**, **MAY** are used in the RFC‑2119 sense.

---

## 1. Transport

CF/x3 rides on A2A unchanged.

### 1.1 Discovery — Agent Card

A server **MUST** serve an A2A *agent card* at:

```
GET {base}/.well-known/agent.json
```

The card declares the three CF/x3 skills and the auth scheme. It **SHOULD** be
reachable without authentication (discovery is public); the data skills are
authenticated. Example:

```json
{
  "name": "Acme Source",
  "description": "CF/x3 context source.",
  "url": "https://acme.example.com/cfx3",
  "version": "1.0.0",
  "capabilities": { "streaming": false, "pushNotifications": false, "stateTransitionHistory": false },
  "authentication": { "schemes": ["bearer"] },
  "defaultInputModes": ["data"],
  "defaultOutputModes": ["data"],
  "skills": [
    { "id": "cfx3.manifest", "name": "Manifest", "description": "Context types + the caller's level on each." },
    { "id": "cfx3.permissions", "name": "Permissions", "description": "Caller subject + per-type access levels." },
    { "id": "cfx3.sync",     "name": "Sync",     "description": "Read context (full or incremental)." },
    { "id": "cfx3.write",    "name": "Write",    "description": "Create/update/delete context nodes." }
  ]
}
```

The card's `url` is the **invocation endpoint** (§1.2).

### 1.2 Invocation — JSON‑RPC `tasks/send`

All three skills are invoked by POSTing a JSON‑RPC 2.0 `tasks/send` request to the
endpoint `url` from the agent card:

```
POST {url}
Content-Type: application/json
Authorization: Bearer <api-key>
```

The request body:

```json
{
  "jsonrpc": "2.0",
  "id": "<client-generated request id>",
  "method": "tasks/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        { "type": "data", "data": { "skill": "<skill-id>", "...": "skill payload" } }
      ]
    }
  }
}
```

- The skill is selected by the `skill` field inside a `data` part.
- The remaining keys of that `data` object are the **skill payload** (§4–§6).
- `text` parts MAY be present but are ignored by CF/x3 skills.

The response is a JSON‑RPC result whose value is an **A2A task** that has run to
completion synchronously. The skill's output rides in the task's first `data`
artifact:

```json
{
  "jsonrpc": "2.0",
  "id": "<same request id>",
  "result": {
    "id": "<server task id>",
    "status": { "state": "completed" },
    "artifacts": [
      { "name": "<skill-id>", "parts": [ { "type": "data", "data": { "...": "skill result" } } ] }
    ]
  }
}
```

A client extracts the result by taking the first `data` part of the first artifact.

On failure, the server returns a standard **JSON‑RPC error response** (A2A‑conformant)
— `{ jsonrpc, id, error: { code, message, data } }` — whose `error.data` is an RFC 9457
problem object. See §7.

> Servers MAY run asynchronously and return `status.state: "working"` with later
> `tasks/get`, per A2A. CF/x3 v1 servers **SHOULD** run synchronously and return
> `completed`; clients **SHOULD** handle a `failed` state by surfacing
> `status.message`.

CF/x3 is a strict **profile of A2A**: it adds well‑known skills + JSON payloads and
does not change A2A's discovery, `tasks/send` transport, task/artifact result shape,
or JSON‑RPC error envelope. An A2A‑conformant client/agent can speak CF/x3.

### 1.3 Authentication & authorization

**Credential:** CF/x3 uses **OAuth 2.0 Bearer tokens**
([RFC 6750](https://www.rfc-editor.org/rfc/rfc6750)). The agent card's
`authentication.schemes` lists `"bearer"`. The client sends
`Authorization: Bearer <token>` on **every** POST. The token MAY be an opaque API
key or a JWT — CF/x3 does not care, as long as the server can resolve it to a
**principal** and that principal's **per‑type permission levels**.

- A server **MUST** reject a missing/invalid token with an RFC 9457 **401
  `unauthenticated`** problem (§7) and SHOULD include `WWW-Authenticate: Bearer`.
  **Every** skill — including `cfx3.manifest` — requires a valid token; there is no
  anonymous access.
- A server **MUST** reject an authenticated caller that lacks the required level for
  an operation with a **403 `insufficient-permission`** problem (§7).

All errors are returned as RFC 9457 Problem Details (§7).

**Permissions are graded per‑type levels.** Each context type has, for a given
caller, exactly **one** access level. Levels are an ordered ladder — each builds on
the previous (if you can `create` you can also `update` and `read`; if you can
`delete` you can also `create`, etc.):

| Level | Name | Grants (cumulative) |
|------:|------|---------------------|
| `0` | `none` | no access — the type is hidden from this caller |
| `1` | `read` | read nodes of the type (`cfx3.sync`) |
| `2` | `update` | + update existing nodes (`cfx3.write` `node.update`) |
| `3` | `create` | + create new nodes (`cfx3.write` `node.create`) |
| `4` | `delete` | + delete nodes (`cfx3.write` `node.delete`) |

A caller with level `L` on a type MAY perform any operation whose **required level**
is `≤ L`. Required levels per operation:

| Operation | Required level |
|-----------|---------------:|
| `cfx3.sync` read of a type | `1` (read) |
| `cfx3.write` `node.update` | `2` (update) |
| `cfx3.write` `node.create` | `3` (create) |
| `cfx3.write` `node.delete` | `4` (delete) |

So one type may be read‑only (level 1), another read+update (2), another up to
delete (4), and another invisible (0). Different callers MAY have different levels on
the same type. **Type management** (creating/deleting context *types*, not nodes) is a
separate source‑level capability, `manageTypes` (§3.3) — most sources have
code‑defined types and set it false.

Servers map their internal roles/ACLs to these levels per request. Clients learn the
caller's level per type from the manifest (§4) and from `cfx3.permissions` (§4.5).

### 1.4 Sessionless

CF/x3 is **stateless / sessionless**. There is no login, handshake, cookie, or
server‑side session. Conformant implementations **MUST**:

- Authenticate and authorize **every request independently** from its bearer token —
  the server derives the principal and per‑type levels per call and keeps no session.
- Keep the **sync cursor client‑owned** (§5): the server does not remember a client's
  position; the client sends `since` and stores the returned `syncedAt`.
- Make each `tasks/send` self‑contained: a request's outcome **MUST NOT** depend on
  prior requests from the same caller (other than durable context state itself).

This makes CF/x3 horizontally scalable and replayable: any server instance can
serve any request, and a client can resume from its stored cursor at any time.

---

## 2. Skills overview

| Skill | Purpose | Read/Write |
|-------|---------|------------|
| `cfx3.manifest` | Declare the context types the caller may access + the caller's level on each | read |
| `cfx3.permissions` | Report the **calling principal's** identity + per‑type access levels | read |
| `cfx3.sync` | Transfer context records (full or incremental) | read |
| `cfx3.write` | Create / update / delete context nodes (and, where allowed, types) | write |

**All four skills require a valid bearer token** (§1.3) — including `cfx3.manifest`.
A server **MUST** implement `cfx3.manifest`, `cfx3.permissions`, and `cfx3.sync`.
`cfx3.write` is **OPTIONAL**; a read‑only source grants no level ≥ 2 and MAY reject
`cfx3.write`.

---

## 3. Data model

All shapes are JSON. Unknown fields **SHOULD** be ignored by receivers
(forward‑compatibility).

### 3.1 Context node — `Cfx3Node`

The unit of federated context.

```ts
interface Cfx3Node {
  uid: string;        // REQUIRED. Stable, source-unique id, e.g. "crm/company/CO123"
  type: string;       // REQUIRED. A context_type name from the manifest
  title: string;      // REQUIRED. Human label
  summary?: string;   // optional short description
  body?: string;      // optional long text (Markdown recommended)
  fields?: Record<string, unknown>;          // typed attributes (per the type's field defs)
  links?: { label: string; target: string }[]; // edges; target is another node's uid
  updated: string;    // REQUIRED. ISO-8601 timestamp of last change at the source
  deleted?: boolean;  // optional; true marks a soft delete (alternative to tombstones)
}
```

**`uid`** is the contract for idempotency. It **MUST** be stable for the life of the
record and unique within the source. A path‑like convention is recommended:
`"<domain>/<type>/<id>"` (e.g. `crm/company/CO123`). Clients treat `(source, uid)`
as the primary key.

**`links[].target`** is another node's `uid`. The receiver resolves it to a local
reference once both nodes are ingested.

### 3.2 Type definition — `Cfx3TypeDef`

Describes a context type the source exposes.

```ts
interface Cfx3TypeDef {
  name: string;          // REQUIRED. unique type id within the source, e.g. "company"
  plural?: string;       // e.g. "companies"
  description?: string;
  baseCategory?: string; // REQUIRED for a root-tier type: the id of a base category
                         // (§3.2.1) this type ultimately rolls up to. Omit only when
                         // `parent` is set (the ancestry is then inherited).
  parent?: string;       // optional. the `name` of a MORE-GENERAL context type this
                         // one specializes (subtype-of). Forms the type tree (§3.2.2).
  fields?: Cfx3FieldDef[];
  level?: number;        // in MANIFEST responses: the caller's access level for this
                         // type, 0–4 (none|read|update|create|delete). Types at level 0
                         // are omitted from the manifest entirely.
}

interface Cfx3FieldDef {
  key: string;           // REQUIRED. matches keys in Cfx3Node.fields
  type: string;          // string|number|boolean|date|datetime|enum|reference|...
  label?: string;
  required?: boolean;
  values?: string[];     // for enum
  targetType?: string;   // for reference (a context type name)
}
```

A type declares its position in the hierarchy with **`baseCategory`** (its tier‑1
root) and/or **`parent`** (a more-specific parent type). See §3.2.2.

#### 3.2.1 Base category — `Cfx3BaseCategory` (tier 1)

Context types are not a flat list: they form a **tree** rooted in a small, stable
set of **base categories** — the top tier of the ontology. (In Phaibel's domain
these are the six *life primitives* a person's world is made of: `person`, `place`,
`thing`, `event`, `task`, `goal`. A source MAY declare a different root set, but
SHOULD keep it small and stable.)

```ts
interface Cfx3BaseCategory {
  id: string;          // REQUIRED. stable id referenced by Cfx3TypeDef.baseCategory, e.g. "person"
  label: string;       // REQUIRED. display name, e.g. "Person"
  description?: string;
}
```

Base categories are the **abstract roots** of the tree — they organize types; nodes
are synced into concrete `context_types`, not directly into a base category.

#### 3.2.2 The context‑type hierarchy

Every context type resolves to exactly one base category, giving a multi‑tier tree:

- **Tier 1 — base categories** (`manifest.base_categories`): the roots.
- **Tier 2 — root types**: a `Cfx3TypeDef` with **no `parent`**; it attaches directly
  to its `baseCategory`.
- **Tier 3+ — subtypes**: a `Cfx3TypeDef` whose `parent` names another type. It
  *inherits* that parent's base‑category ancestry (so `baseCategory` MAY be omitted on
  a subtype; if present it MUST agree with the resolved root).

A receiver reconstructs the tree by: for each type, attach under `parent` if set,
else under `baseCategory`; roots are the `base_categories`. The **depth** below the
base category is the type's *specificity* — a more‑specific subtype (e.g.
`enterprise-customer` → `customer` → `thing`) is a stronger signal than a generic one.

Rules:
- `parent` MUST reference a `name` present in the same manifest's `context_types`
  (no dangling parents); cycles are invalid.
- A type with neither `parent` nor `baseCategory` is malformed; receivers SHOULD
  treat it as a tier‑2 type under an implicit/unknown root and MAY warn.

### 3.3 Permissions — `Cfx3Permissions`

The calling principal's identity and per‑type access levels — returned by
[`cfx3.permissions`](#45-cfx3permissions--caller-permissions). Derived per request
from the bearer token (sessionless).

```ts
type Cfx3Level = 0 | 1 | 2 | 3 | 4;   // none | read | update | create | delete (cumulative)

interface Cfx3Permissions {
  subject: string;                    // stable, opaque id of the calling principal (OAuth `sub`)
  manageTypes: boolean;               // may create/update/delete context TYPES
  types: Record<string, Cfx3Level>;   // type name → the caller's level (0–4)
}
```

A `types` entry of `0` (or an absent entry) means no access to that type.

### 3.4 Manifest — `Cfx3Manifest`

```ts
interface Cfx3Manifest {
  cfx3_version: number;             // protocol version (1)
  source: string;                  // stable source id, e.g. "synaptic"
  name: string;                    // display name
  base_categories: Cfx3BaseCategory[]; // tier-1 roots of the type tree (§3.2.1)
  context_types: Cfx3TypeDef[];     // the types, each with baseCategory and/or parent
                                    // forming a tree (§3.2.2); only types the caller can
                                    // read (level ≥ 1), each annotated with `level`
  manageTypes: boolean;             // whether the caller may manage context types
  auth: { schemes: ("bearer")[] };
}
```

The manifest **describes the source's context‑type tree**: `base_categories` are the
roots (tier 1) and each `Cfx3TypeDef` declares its place via `baseCategory`/`parent`
(§3.2.2). It is also **permission‑scoped to the caller**: it lists only types the
caller may read (level ≥ 1) and annotates each with the caller's `level`. When
permission scoping hides a parent type, its visible subtypes fall back to their
resolved `baseCategory` root. `cfx3.permissions` returns the same per‑type levels in a
compact map (§4.5).

---

## 4. `cfx3.manifest`

Returns the source's [`Cfx3Manifest`](#34-manifest--cfx3manifest).

**Request payload:** *(none)* — `{ "skill": "cfx3.manifest" }`.

**Result:** the manifest object.

### 4.1 Purpose & protection

The manifest tells the caller *what context it can see* and *at what level*. It is
**protected**: the server **MUST** require a valid bearer token (401 otherwise) and
**MUST** return only the types the caller may read. A client **SHOULD** fetch it
before its first sync and refresh it periodically (e.g. daily) or on demand, and
**SHOULD** ensure its local store has matching context types before ingesting.

### 4.2 Permission scoping

A server **MUST** omit any context type the caller cannot read (level 0), and
**MUST** annotate each returned type with the caller's `level` (1–4). The client uses
`level` to decide which operations to offer for that type:

| `level` | sync (read) | update | create | delete |
|--------:|:-----------:|:------:|:------:|:------:|
| 1 read | ✓ | | | |
| 2 update | ✓ | ✓ | | |
| 3 create | ✓ | ✓ | ✓ | |
| 4 delete | ✓ | ✓ | ✓ | ✓ |

---

## 4.5 `cfx3.permissions` — caller permissions

Returns the **calling principal's** access levels, derived per request from the
bearer token (sessionless). This is how a client learns who it is and what it may do
across all types in one call.

**Request payload:** *(none)* — `{ "skill": "cfx3.permissions" }`.

**Result:** a [`Cfx3Permissions`](#33-permissions--cfx3permissions) object.

- `subject` is a stable, opaque id for the principal (the OAuth `sub`); useful for
  attributing writes and for the client to detect a credential change.
- `manageTypes` indicates whether the caller may create/update/delete context types.
- `types` maps each type name to the caller's level (0–4). It is the authoritative,
  compact form of the same per‑type levels the manifest annotates.

A server **MUST** compute this from the request's token only (no session). A client
**SHOULD** call it on connect and after a manifest refresh, and use it to gate
write/delete affordances.

---

## 5. `cfx3.sync` — read context

Transfers context records. The **client owns the cursor**.

**Request payload — `Cfx3SyncRequest`:**

```ts
interface Cfx3SyncRequest {
  since?: string | null;   // null/absent ⇒ FULL sync; ISO-8601 ⇒ records changed since then
  types?: string[];        // optional: restrict to these context types
  limit?: number;          // optional: server page-size hint
  cursor?: string;         // optional: opaque paging cursor from a prior nextCursor
}
```

**Result — `Cfx3SyncResult`:**

```ts
interface Cfx3SyncResult {
  records: Cfx3Node[];     // created/updated nodes
  tombstones: string[];    // uids deleted since `since`
  syncedAt: string;        // ISO-8601 — the client stores this as its next `since`
  nextCursor?: string;     // present when more pages remain for this sync
}
```

### 5.1 Cursor semantics

- A **full sync** is `since: null` (or omitted): the server returns all records the
  caller may read.
- An **incremental sync** passes the `syncedAt` value from the previous response as
  `since`. The server returns records whose `updated > since` plus tombstones for
  records deleted since `since`.
- The server **MUST** return a fresh `syncedAt` each call (typically "now"). The
  client **MUST** persist it and send it as the next `since`. The client owns and
  carries the cursor; the server is stateless with respect to it.

### 5.2 Paging

If a result is large, the server **MAY** return a `nextCursor` and the client
**SHOULD** re‑issue `cfx3.sync` with that `cursor` (keeping the same `since`) until
no `nextCursor` is returned, then store the final `syncedAt`.

### 5.3 Deletes

Two mechanisms (a server MAY use either):

- **Tombstones:** list deleted `uid`s in `tombstones` for the window since `since`.
  This requires the server to track deletions (e.g. a `deleted_at`); if it cannot,
  it returns `tombstones: []`.
- **Soft‑delete flag:** include the record with `deleted: true`.

Clients **MUST** remove (or trash) local nodes whose `uid` appears in `tombstones`
or arrives with `deleted: true`.

---

## 6. `cfx3.write` — create / update / delete context

The only mutation surface, and strictly context. Domain actions are out of scope
(use MCP).

**Request payload — `Cfx3WriteRequest`:**

```ts
type Cfx3WriteOp =
  | "node.create" | "node.update" | "node.delete"
  | "type.create" | "type.update" | "type.delete";

interface Cfx3WriteRequest {
  op: Cfx3WriteOp;                 // REQUIRED
  node?: {                         // node.create / node.update
    type?: string;                 // required for node.create
    uid?:  string;                 // required for node.update
    title?: string;
    fields?: Record<string, unknown>;
    body?:  string;
    links?: { label: string; target: string }[];
  };
  uid?: string;                    // node.delete
  type?: Cfx3TypeDef;              // type.create / type.update
  typeName?: string;              // type.delete
}
```

**Result — `Cfx3WriteResult`** (on success). Failures are returned as RFC 9457
problems (§7), not as this shape.

```ts
interface Cfx3WriteResult {
  ok: true;
  uid: string;      // uid of the created/updated node (for delete: the deleted uid)
}
```

### 6.1 Operation semantics

- **`node.create`** — `node.type` and `node.title` **MUST** be present. The server
  assigns the `uid` and returns it. The server validates `fields` against the type
  (a violation is a `422` problem, §7).
- **`node.update`** — `node.uid` **MUST** be present; provided fields are merged.
- **`node.delete`** — `uid` **MUST** be present.
- **`type.create` / `type.update`** — supply a `Cfx3TypeDef` in `type`. Allowed only
  when `manageTypes` is true; many sources have code‑defined types and reject these.
- **`type.delete`** — `typeName` **MUST** be present; same capability rule.

### 6.2 Permissions

The server **MUST** enforce the operation's **required level** server‑side
(`node.update` ≥ 2, `node.create` ≥ 3, `node.delete` ≥ 4; type ops require
`manageTypes`), regardless of what any cached manifest advertised. A write the caller
lacks the level for returns an `insufficient-permission` problem (status 403),
delivered as a JSON‑RPC error per §7.

### 6.3 Idempotency note for write‑back

When a client writes a node it originally mirrored from a source, it **SHOULD** use
`node.update` with the node's existing `uid` (the value it stored as `sourceId`), so
the source updates in place rather than creating a duplicate.

---

## 7. Errors — RFC 9457 Problem Details, carried in A2A JSON‑RPC

CF/x3 stays **fully A2A**: errors are returned as **JSON‑RPC 2.0 error responses**
(the A2A envelope). The error body follows **RFC 9457** *Problem Details for HTTP
APIs* ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) by carrying a problem
object in the JSON‑RPC **`error.data`** member. This gives a standard, machine‑
readable error format without leaving A2A/JSON‑RPC.

On error the server returns:

```json
{
  "jsonrpc": "2.0",
  "id": "<request id>",
  "error": {
    "code": -32040,
    "message": "Insufficient permission",
    "data": {
      "type": "https://cfx3.dev/problems/insufficient-permission",
      "title": "Insufficient permission",
      "status": 403,
      "detail": "Level 1 (read) cannot create 'company' (requires 3).",
      "instance": "urn:cfx3:req:7af3",
      "contextType": "company", "requiredLevel": 3, "yourLevel": 1
    }
  }
}
```

- `error.data` **MUST** be an RFC 9457 problem object: `type` (URI), `title`,
  `status`, optional `detail`/`instance`, plus extension members. Clients **SHOULD**
  branch on `data.type`, not on text.
- `error.code` is the JSON‑RPC code (below); `error.message` mirrors `data.title`.
- `data.status` carries the HTTP‑equivalent status (401/403/404/422/500).

### 7.1 HTTP status

JSON‑RPC over HTTP normally returns **HTTP 200** with the error in the body, and CF/x3
follows that for in‑band errors. The one exception A2A permits is **authentication**:
a server **MAY** reject a missing/invalid token at the HTTP layer with **401** +
`WWW-Authenticate: Bearer` (body SHOULD still be the JSON‑RPC error with the
`unauthenticated` problem). Either way, clients learn the precise cause from
`error.data.type` / `data.status`.

### 7.2 Code ↔ problem registry (DRAFT)

`type` URIs are under `https://cfx3.dev/problems/` (canonical registry; treat as
opaque identifiers). CF/x3 uses standard JSON‑RPC codes for envelope problems and the
application code **`-32040`** for semantic ones (chosen to avoid colliding with A2A's
own task error codes, e.g. `TaskNotFound -32001`).

| JSON‑RPC `code` | problem `type` (slug) | `status` | When | Extensions |
|----------------:|-----------------------|---------:|------|------------|
| `-32700` | `invalid-request` | 400 | malformed JSON | |
| `-32600` | `invalid-request` | 400 | missing/invalid request | |
| `-32601` | `invalid-request` | 400 | method ≠ `tasks/send` | |
| `-32602` | `unknown-skill` | 400 | unknown `data.skill` / bad params | `skill` |
| `-32040` | `unauthenticated` | 401 | missing/invalid/expired token | |
| `-32040` | `insufficient-permission` | 403 | caller's level too low | `contextType`, `requiredLevel`, `yourLevel` |
| `-32040` | `type-management-unsupported` | 403 | `type.*` when `manageTypes` false | |
| `-32040` | `not-found` | 404 | unknown `uid`/type on update/delete | `uid` |
| `-32040` | `unprocessable` | 422 | field/type validation failed | `errors` (per‑field) |
| `-32603` | `server-error` | 500 | unexpected failure | |

Unspecified errors **SHOULD** use `type: "about:blank"` with an appropriate `status`.

### 7.3 Client handling

Clients read the JSON‑RPC `error`, then `error.data`: branch on `data.type` (refresh
the token on `unauthenticated`, hide an affordance on `insufficient-permission`,
surface `detail`), falling back to `error.code`/`error.message` if `data` is absent.
Also honor a transport‑level **401** for auth.

---

## 8. Idempotency & provenance (client obligations)

A conformant client **MUST**:

1. Treat **`(source, uid)`** as the primary key for mirrored nodes. On ingest, look
   up an existing local node with the same source + remote uid; **update** it if
   found, otherwise **create** it. Re‑syncing the same record **MUST NOT** create a
   duplicate.
2. **Record provenance** on every mirrored node: the source id and the remote `uid`
   (commonly stored as `source` and `sourceId`). This lets the client re‑sync,
   diff, and cleanly detach a source.
3. Apply **tombstones / `deleted`** by removing the corresponding local node.
4. Persist `syncedAt` per source as the next `since`.

---

## 9. Versioning

`cfx3_version` is an integer in the manifest. v1 is this document. A client
**SHOULD** read it and refuse (or degrade) if it does not support the server's
version. Additive, backward‑compatible changes keep the same version; breaking
changes bump it.

---

## 10. Examples

### 10.1 Manifest

Request:

```json
{ "jsonrpc": "2.0", "id": "1", "method": "tasks/send",
  "params": { "message": { "role": "user", "parts": [ { "type": "data", "data": { "skill": "cfx3.manifest" } } ] } } }
```

Result (artifact data):

```json
{
  "cfx3_version": 1,
  "source": "synaptic",
  "name": "Synaptic",
  "base_categories": [
    { "id": "person", "label": "Person" },
    { "id": "place",  "label": "Place" },
    { "id": "thing",  "label": "Thing" },
    { "id": "event",  "label": "Event" },
    { "id": "task",   "label": "Task" },
    { "id": "goal",   "label": "Goal" }
  ],
  "context_types": [
    { "name": "company", "plural": "companies", "baseCategory": "place", "level": 4,
      "fields": [ { "key": "domain", "type": "string" }, { "key": "industry", "type": "string" } ] },
    { "name": "contact", "plural": "contacts", "baseCategory": "person", "level": 2,
      "fields": [ { "key": "email", "type": "string" }, { "key": "role", "type": "string" } ] },
    { "name": "customer", "plural": "customers", "baseCategory": "thing", "level": 1,
      "fields": [ { "key": "billingEmail", "type": "string" } ] },
    { "name": "enterprise-customer", "plural": "enterprise customers", "parent": "customer", "level": 1,
      "fields": [ { "key": "seats", "type": "number" } ] },
    { "name": "account", "plural": "accounts", "baseCategory": "thing", "level": 1,
      "fields": [ { "key": "code", "type": "string" } ] }
  ],
  "manageTypes": false,
  "auth": { "schemes": ["bearer"] }
}
```

The tree this describes:

```
place ──▶ company
person ─▶ contact
thing ──▶ customer ──▶ enterprise-customer   (subtype via parent)
         ▶ account
```

`enterprise-customer` omits `baseCategory` — it inherits `thing` through its `parent`
`customer`, and is one tier more specific. Permission‑wise the caller may delete
companies (4), update contacts (2), and only read customers/accounts (1). Any type
they can't read (level 0) is absent.

### 10.2 Permissions (who am I)

Request: `{ "skill": "cfx3.permissions" }`. Result (artifact data):

```json
{
  "subject": "user_8f3a",
  "manageTypes": false,
  "types": { "company": 4, "contact": 2, "account": 1, "deal": 0 }
}
```

### 10.3 Sync (incremental)

Request:

```json
{ "jsonrpc": "2.0", "id": "2", "method": "tasks/send",
  "params": { "message": { "role": "user", "parts": [
    { "type": "data", "data": { "skill": "cfx3.sync", "since": "2026-06-20T14:00:00.000Z" } } ] } } }
```

Result (artifact data):

```json
{
  "records": [
    { "uid": "crm/company/CO8h2k1", "type": "company", "title": "Acme Corp",
      "updated": "2026-06-26T18:03:11.000Z",
      "fields": { "domain": "acme.com", "industry": "Manufacturing" },
      "links": [ { "label": "customer", "target": "sub/customer/CU3a9" } ] },
    { "uid": "crm/contact/CN5p7", "type": "contact", "title": "Dana Reyes",
      "updated": "2026-06-26T17:55:00.000Z",
      "fields": { "email": "dana@acme.com", "role": "economic_buyer" },
      "links": [ { "label": "works-at", "target": "crm/company/CO8h2k1" } ] }
  ],
  "tombstones": [ "crm/deal/DE0aa1" ],
  "syncedAt": "2026-06-26T18:10:42.000Z"
}
```

The client upserts both records keyed on `(synaptic, uid)`, trashes the tombstoned
node, and stores `2026-06-26T18:10:42.000Z` as the next `since`.

### 10.4 Write (create a context node)

Request:

```json
{ "jsonrpc": "2.0", "id": "3", "method": "tasks/send",
  "params": { "message": { "role": "user", "parts": [
    { "type": "data", "data": {
      "skill": "cfx3.write",
      "op": "node.create",
      "node": {
        "type": "contact",
        "title": "Sam Lee",
        "fields": { "email": "sam@acme.com", "company_suid": "CO8h2k1" },
        "links": [ { "label": "works-at", "target": "crm/company/CO8h2k1" } ]
      }
    } } ] } } }
```

Result — HTTP 200, JSON‑RPC result whose artifact data is:

```json
{ "ok": true, "uid": "crm/contact/CT9k2p1" }
```

A rejected write — a JSON‑RPC **error** response carrying an RFC 9457 problem (§7):

```json
{ "jsonrpc": "2.0", "id": "3",
  "error": { "code": -32040, "message": "Insufficient permission",
    "data": {
      "type": "https://cfx3.dev/problems/insufficient-permission",
      "title": "Insufficient permission", "status": 403,
      "detail": "Level 1 (read) cannot create 'contact' (requires 3).",
      "contextType": "contact", "requiredLevel": 3, "yourLevel": 1 } } }
```

---

## 11. Conformance checklist

**Server**
- [ ] `GET /.well-known/agent.json` returns a card with the four skills + `auth`.
- [ ] `POST {url}` handles JSON‑RPC `tasks/send`; routes on `data.skill`; results are
      A2A tasks with a `data` artifact (stays A2A‑conformant).
- [ ] Authenticates the **bearer token per request** (sessionless), on **every**
      skill including `cfx3.manifest`.
- [ ] Maps internal roles → a **per‑type level (0–4)**; enforces the op's required level.
- [ ] `cfx3.manifest` returns `base_categories` (tier‑1 roots) and the context‑type
      tree (each type with `baseCategory` and/or `parent`); only types with level ≥ 1,
      each annotated with `level`; no dangling `parent` references.
- [ ] `cfx3.permissions` returns the caller's `subject`, `manageTypes`, and `types` level map.
- [ ] `cfx3.sync` supports full (`since:null`) and incremental; returns `records`,
      `tombstones`, `syncedAt`; keeps **no cursor state**.
- [ ] (If writable) `cfx3.write` returns `{ ok:true, uid }` on success.
- [ ] **Errors** are JSON‑RPC error responses with an **RFC 9457** problem in
      `error.data` (§7); auth MAY also use transport 401.

**Client**
- [ ] Sends `Authorization: Bearer` on every call; assumes no session.
- [ ] Fetches + caches the manifest and `cfx3.permissions`; ensures local types exist,
      preserving the hierarchy (`base_categories` roots + each type's `baseCategory`/`parent`).
- [ ] Gates each operation on the type's `level` (read ≥1, update ≥2, create ≥3, delete ≥4).
- [ ] Sends `since` cursor; persists returned `syncedAt`; supports paging.
- [ ] Upserts keyed on `(source, uid)`; records provenance; applies tombstones.
- [ ] Handles errors by branching on `error.data.type` (RFC 9457); honors 401.
- [ ] (If writing back) uses `node.update` with the stored `uid` for existing nodes.
