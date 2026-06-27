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
    { "id": "cfx3.manifest", "name": "Manifest", "description": "Context types + write capabilities." },
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

> Servers MAY run asynchronously and return `status.state: "working"` with later
> `tasks/get`, per A2A. CF/x3 v1 servers **SHOULD** run synchronously and return
> `completed`; clients **SHOULD** handle a `failed` state by surfacing
> `status.message`.

### 1.3 Authentication

The agent card's `authentication.schemes` lists accepted schemes. v1 defines
`"bearer"`: the client sends `Authorization: Bearer <api-key>`. A server **MAY**
require auth on all skills and **MUST** reject unauthorized calls with the
transport's standard 401, or a JSON‑RPC error (§7). Servers **SHOULD** scope what
a credential can see and do (see capabilities, §4.3, and per‑type filtering, §4.2).

---

## 2. Skills overview

| Skill | Purpose | Read/Write |
|-------|---------|------------|
| `cfx3.manifest` | Declare the context types the source exposes + write capabilities | read |
| `cfx3.sync` | Transfer context records (full or incremental) | read |
| `cfx3.write` | Create / update / delete context nodes (and, where allowed, types) | write |

A server **MUST** implement `cfx3.manifest` and `cfx3.sync`. `cfx3.write` is
**OPTIONAL**; a read‑only source advertises no write capabilities (§4.3) and MAY
reject `cfx3.write`.

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
  name: string;          // REQUIRED. e.g. "company"
  plural?: string;       // e.g. "companies"
  description?: string;
  baseCategory?: string; // optional ontology hint: person|place|thing|event|task|goal
  fields?: Cfx3FieldDef[];
  readonly?: boolean;    // true ⇒ nodes of this type cannot be created/updated/deleted
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

`baseCategory` is an optional hint to help consumers slot the type into their own
ontology; consumers MAY ignore it.

### 3.3 Capabilities — `Cfx3Capabilities`

What write operations the **caller** may perform (after the server applies the
caller's permissions). See §4.3.

```ts
interface Cfx3Capabilities {
  writeNodes: boolean;   // node.create / node.update allowed
  deleteNodes: boolean;  // node.delete allowed
  manageTypes: boolean;  // type.create / type.update / type.delete allowed
}
```

### 3.4 Manifest — `Cfx3Manifest`

```ts
interface Cfx3Manifest {
  cfx3_version: number;          // protocol version (1)
  source: string;               // stable source id, e.g. "synaptic"
  name: string;                 // display name
  context_types: Cfx3TypeDef[];
  capabilities: Cfx3Capabilities;
  auth: { schemes: ("bearer")[] };
}
```

---

## 4. `cfx3.manifest`

Returns the source's [`Cfx3Manifest`](#34-manifest--cfx3manifest).

**Request payload:** *(none)* — `{ "skill": "cfx3.manifest" }`.

**Result:** the manifest object.

### 4.1 Purpose

The manifest is the source of truth for *what context exists* and *what the caller
may do*. A client **SHOULD** fetch it before its first sync and refresh it
periodically (e.g. daily) or on demand. A client **SHOULD** ensure its local store
has matching context types before ingesting (creating them from `context_types`).

### 4.2 Permission filtering

The manifest **SHOULD** reflect the caller's permissions: a server **SHOULD** omit
context types the caller cannot read, and set `readonly: true` on types the caller
cannot write.

### 4.3 Capabilities

`capabilities` declares the caller's allowed write operations *for this source as a
whole*; per‑type `readonly` narrows it further. Clients **SHOULD** use these to
decide whether to offer write/delete affordances.

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

**Result — `Cfx3WriteResult`:**

```ts
interface Cfx3WriteResult {
  ok: boolean;
  uid?: string;     // uid of the created/updated node
  message?: string; // failure reason when ok=false
}
```

### 6.1 Operation semantics

- **`node.create`** — `node.type` and `node.title` **MUST** be present. The server
  assigns the `uid` and returns it. The server validates `fields` against the type.
- **`node.update`** — `node.uid` **MUST** be present; provided fields are merged.
- **`node.delete`** — `uid` **MUST** be present.
- **`type.create` / `type.update`** — supply a `Cfx3TypeDef` in `type`. Allowed only
  when `capabilities.manageTypes` is true; many sources have code‑defined types and
  return `ok: false`.
- **`type.delete`** — `typeName` **MUST** be present; same capability rule.

### 6.2 Permissions

The server **MUST** enforce write permission server‑side regardless of what the
manifest advertised. A rejected write returns `ok: false` with a `message` (the
transport status MAY remain 200; surface the reason from `message`).

### 6.3 Idempotency note for write‑back

When a client writes a node it originally mirrored from a source, it **SHOULD** use
`node.update` with the node's existing `uid` (the value it stored as `sourceId`), so
the source updates in place rather than creating a duplicate.

---

## 7. Errors

CF/x3 uses JSON‑RPC error objects for transport/dispatch problems, and the
`ok/message` field for write‑level rejections.

```json
{ "jsonrpc": "2.0", "id": "<id>", "error": { "code": -32602, "message": "Unknown CF/x3 skill: …" } }
```

Recommended codes:

| Code | Meaning |
|------|---------|
| `-32700` | Parse error (malformed JSON) |
| `-32600` | Invalid request (missing method) |
| `-32601` | Unsupported method (only `tasks/send`) |
| `-32602` | Invalid params / unknown skill |
| `-32000` | Server error while executing a skill |

Auth failures **SHOULD** use the transport 401. A `cfx3.write` that is understood
but not permitted **SHOULD** return `result.artifacts[0].data = { ok: false,
message }` rather than a JSON‑RPC error.

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
  "context_types": [
    { "name": "company", "plural": "companies", "baseCategory": "place",
      "fields": [ { "key": "domain", "type": "string" }, { "key": "industry", "type": "string" } ] },
    { "name": "contact", "plural": "contacts", "baseCategory": "person",
      "fields": [ { "key": "email", "type": "string" }, { "key": "role", "type": "string" } ] },
    { "name": "account", "plural": "accounts", "baseCategory": "thing", "readonly": true,
      "fields": [ { "key": "code", "type": "string" } ] }
  ],
  "capabilities": { "writeNodes": true, "deleteNodes": false, "manageTypes": false },
  "auth": { "schemes": ["bearer"] }
}
```

### 10.2 Sync (incremental)

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

### 10.3 Write (create a context node)

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

Result (artifact data):

```json
{ "ok": true, "uid": "crm/contact/CT9k2p1" }
```

A rejected write:

```json
{ "ok": false, "message": "forbidden: write permission required" }
```

---

## 11. Conformance checklist

**Server**
- [ ] `GET /.well-known/agent.json` returns a card with the three skills + `auth`.
- [ ] `POST {url}` handles JSON‑RPC `tasks/send`; routes on `data.skill`.
- [ ] `cfx3.manifest` returns a permission‑filtered manifest with `cfx3_version`.
- [ ] `cfx3.sync` supports full (`since:null`) and incremental; returns `records`,
      `tombstones`, `syncedAt`.
- [ ] (If writable) `cfx3.write` enforces permissions and returns `{ ok, uid?, message? }`.
- [ ] Returns JSON‑RPC errors per §7; 401 on auth failure.

**Client**
- [ ] Fetches + caches the manifest; ensures local context types exist.
- [ ] Sends `since` cursor; persists returned `syncedAt`; supports paging.
- [ ] Upserts keyed on `(source, uid)`; records provenance; applies tombstones.
- [ ] (If writing back) uses `node.update` with the stored `uid` for existing nodes.
