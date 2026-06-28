# Building a CF/x3 Server

> **⚠️ DRAFT.** Working draft; subject to change. Read the
> [Protocol specification](02-protocol.md) first — this guide is implementation
> advice, the spec is normative.

A **CF/x3 server** exposes part of your system as federated context: it declares
which **context types** it offers, serves **reads** (full + incremental), and
optionally accepts **writes** (CRUD on context nodes). It does **not** expose
domain actions — keep those in a separate MCP server.

This guide is language‑agnostic; pseudocode is illustrative.

## 1. Decide your context model

Map your entities to **context types** and each row/record to a **context node**.

- Pick a stable, unique **`uid`** scheme per node. A path form works well:
  `"<domain>/<type>/<recordId>"`, e.g. `crm/company/CO123`. The `uid` MUST be stable
  for the record's lifetime — clients use `(source, uid)` as a primary key.
- For each type, list the **fields** you'll expose (key + type). A type the caller
  can't mutate simply has a max **level** of `1` (read) for that caller (§7).
- Use **`links`** for relationships, with `target` set to another node's `uid`
  (e.g. a contact links `works-at` → `crm/company/CO123`).

Example mapping (a CRM):

| Your table | Context type | uid | fields |
|------------|-------------|-----|--------|
| companies | `company` | `crm/company/{id}` | domain, industry, city, … |
| contacts | `contact` | `crm/contact/{id}` | email, role, … (+ link to company) |
| ledger accounts | `account` (read‑only → level 1) | `acc/account/{id}` | code, type, … |

## 2. Serve the agent card

```
GET {base}/.well-known/agent.json   →  the A2A card (see spec §1.1)
```

It declares the three skills and `authentication.schemes`. Keep discovery public;
gate the data skills.

## 3. The JSON‑RPC endpoint

```
POST {url}   (the `url` from your agent card)
```

Dispatch on the `skill` field of the request's `data` part:

CF/x3 is **sessionless**: authenticate and authorize **every request** from its
bearer token — no session, no login step. Resolve the token to a principal and its
**per‑type levels** per call.

```
function handlePost(req):
    body = parseJson(req)                       # else JSON-RPC -32700
    if body.method != "tasks/send": return rpcError(-32601)
    data  = firstDataPart(body.params.message.parts) or {}
    skill = data.skill

    principal = authenticate(req.bearerToken)   # 401 if missing/invalid (sessionless)
    if principal is null: return http401()
    levels = levelsFor(principal)               # map your RBAC → per-type level (per request)

    switch skill:
      "cfx3.manifest":    return rpcResult(body.id, skill, buildManifest(levels))
      "cfx3.permissions": return rpcResult(body.id, skill, buildPermissions(principal, levels))
      "cfx3.sync":        return rpcResult(body.id, skill, runSync(levels, data.since, data.types, data.cursor))
      "cfx3.write":       return rpcResult(body.id, skill, runWrite(levels, data))  # raises an RFC 9457 problem (§8) if level too low
      default:            return rpcError(body.id, -32602, "Unknown CF/x3 skill")
```

Note every skill (including `cfx3.manifest`) is behind the auth check — there is no
anonymous access.

### Map your RBAC → per‑type levels

The protocol expresses permission as a **level 0–4 per context type**
(`0 none · 1 read · 2 update · 3 create · 4 delete`, cumulative). Translate your
roles to a level per type, per request:

```
LEVEL = { none:0, read:1, update:2, create:3, delete:4 }
REQUIRED = { "sync":1, "node.update":2, "node.create":3, "node.delete":4 }

function levelsFor(principal):              # → { "<type>": 0..4 }
    m = {}
    for type in TYPES:
        if    principal.isAdminOf(type):  m[type] = LEVEL.delete    # 4
        elif  principal.canCreate(type):  m[type] = LEVEL.create    # 3
        elif  principal.canEdit(type):    m[type] = LEVEL.update     # 2
        elif  principal.canRead(type):    m[type] = LEVEL.read       # 1
        else:                             m[type] = LEVEL.none       # 0
    return m

allowed(levels, op, type) = levels[type] >= REQUIRED[op]   # monotonic ladder
```

Where `rpcResult` wraps the payload in a completed A2A task artifact:

```
rpcResult(id, skill, payload) =
  { jsonrpc:"2.0", id, result:{ id: newId(), status:{state:"completed"},
      artifacts:[ { name: skill, parts:[ { type:"data", data: payload } ] } ] } }
```

## 4. `cfx3.manifest`

Require a valid token (401 otherwise). Return only types the caller can **read**
(level ≥ 1), each annotated with the caller's `level`:

```
buildManifest(levels) =
  { cfx3_version: 1,
    source: "<your-source-id>",
    name:   "<display name>",
    context_types: TYPES
        .filter(t => levels[t.name] >= 1)
        .map(t => ({ ...stripInternalFields(t), level: levels[t.name] })),
    manageTypes: principalCanManageTypes,    # usually false (code-defined types)
    auth: { schemes: ["bearer"] } }
```

## 4a. `cfx3.permissions`

Report the **caller's** identity and per‑type level map — computed from the request
token only (sessionless):

```
buildPermissions(principal, levels) =
  { subject: principal.stableId,           # opaque, stable (OAuth `sub`)
    manageTypes: principalCanManageTypes,
    types: levels }                        # { "<type>": 0..4 }
```

Clients call this to decide which read/update/create/delete affordances to show.

## 5. `cfx3.sync`

Translate each record into a `Cfx3Node`. Honor the cursor and type filter.

```
runSync(levels, since, types, cursor):
    records = []
    for type in TYPES where levels[type.name] >= 1 and (types is empty or type in types):
        rows = query(type, where: since ? "updated_at > :since" : "true",
                     page: cursor, limit: PAGE)
        for row in rows:
            records.push(toNode(type, row))     # uid, type, title, fields, links, updated
    tombstones = since ? deletedUidsSince(since, levels) : []   # [] if you don't track deletes
    return { records, tombstones, syncedAt: nowIso(), nextCursor: morePages ? next : undefined }
```

Notes:
- **Full sync** when `since` is null/absent; **incremental** filters on your
  `updated_at`. Always return a fresh `syncedAt` (typically "now").
- **Deletes:** if you track deletions (e.g. `deleted_at`), return their `uid`s in
  `tombstones`; otherwise return `[]`. Alternatively emit the record with
  `deleted: true`.
- **Paging:** for large sets, return `nextCursor`; the client re‑calls with the same
  `since` until it's gone.
- `toNode` is your reverse mapping: assign the `uid`, set `title`, put attributes in
  `fields`, and express relationships as `links` (targets are other `uid`s).

## 6. `cfx3.write` (optional)

Only context CRUD. Enforce permissions **server‑side** — never trust the manifest
the client saw.

```
runWrite(levels, data):
    op = data.op
    if op starts with "type." and not principalCanManageTypes:
        return { ok:false, message:"type management not supported" }

    if op == "node.delete":
        (type, id) = parseUid(data.uid)
        if levels[type] < REQUIRED["node.delete"]: return { ok:false, message:"insufficient_permission (need delete) on "+type }
        deleteRecord(type, id); return { ok:true, uid:data.uid }

    if op in ("node.create","node.update"):
        node = data.node
        type = node.type or parseUid(node.uid).type
        if levels[type] < REQUIRED[op]: return { ok:false, message:"insufficient_permission on "+type }
        cols = mapFieldsToColumns(type, node.fields, node.title, node.links)
        if op == "node.update":
            id = parseUid(node.uid).id; updateRecord(type, id, cols); return { ok:true, uid:node.uid }
        else:
            id = insertRecord(type, withRequiredDefaults(type, cols)); return { ok:true, uid: makeUid(type, id) }

    return { ok:false, message:"unknown op" }
```

Guidelines:
- `node.create` MUST assign and return the `uid`. `node.update` keys on `node.uid`.
- Validate `fields` against the type; apply your own required‑field defaults on
  create (owner, timestamps, default status, …).
- Resolve a `links` target like `crm/company/CO123` back to your foreign key when
  the field isn't given directly.
- A read‑only type is just one whose max level is 1 — the level check already rejects
  writes to it.

## 7. Permissions (per‑type levels)

- Authenticate the bearer token to a **principal** on every request (sessionless),
  including for `cfx3.manifest` — no anonymous access.
- Map the principal's roles/ACLs to a **level 0–4 per context type** (`levelsFor`, §3).
  The ladder is cumulative: `read(1) ⊂ update(2) ⊂ create(3) ⊂ delete(4)`.
- Use levels everywhere: filter the manifest to level ≥ 1, annotate each type's
  `level`, answer `cfx3.permissions`, gate `sync` (≥1) and each `write` op
  (update ≥2, create ≥3, delete ≥4). Enforce **server‑side** regardless of any
  manifest the client cached.

## 8. Errors — A2A JSON‑RPC + RFC 9457

Stay A2A: return a **JSON‑RPC error response** whose **`error.data`** is an **RFC 9457**
problem object (spec §7). One helper:

```
rpcError(id, code, problem) =
  { jsonrpc:"2.0", id, error:{ code, message: problem.title, data: problem } }

problem(slug, status, detail, extra={}) =
  { type:"https://cfx3.dev/problems/"+slug, title: titleFor(slug), status, detail, ...extra }
```

Map:

| Situation | `code` | problem slug / status |
|-----------|-------:|------------------------|
| malformed JSON / not `tasks/send` | -32700/-32601 | `invalid-request` 400 |
| unknown `data.skill` / bad params | -32602 | `unknown-skill` 400 |
| missing/invalid token | -32040 (or HTTP **401**) | `unauthenticated` 401 (+`WWW-Authenticate: Bearer`) |
| level too low | -32040 | `insufficient-permission` 403 (+`contextType`/`requiredLevel`/`yourLevel`) |
| `type.*` when types are code‑defined | -32040 | `type-management-unsupported` 403 |
| unknown `uid`/type on update/delete | -32040 | `not-found` 404 (+`uid`) |
| field validation failed | -32040 | `unprocessable` 422 (+`errors`) |
| unexpected | -32603 | `server-error` 500 |

So `runWrite` returns successes as `{ ok:true, uid }` in a JSON‑RPC **result**, and
failures as a JSON‑RPC **error** carrying the problem (replace the illustrative
`{ ok:false, message }` returns above with `rpcError(...)`). Authentication MAY be
enforced at the HTTP layer (401) per A2A.

## 9. Keep actions out

If your system also has domain actions (state transitions, sends, charges), expose
them via a **separate MCP server**, not CF/x3. CF/x3 stays context‑only so clients
can mirror you safely. See [Why context federation](01-why-context-federation.md).

## 10. Test your server

- `curl {base}/.well-known/agent.json` → card with 3 skills.
- POST `cfx3.manifest` with a bearer key → context types, each with your `level`.
- POST `cfx3.permissions` → your `subject` + per‑type level map.
- POST `cfx3.sync {since:null}` → records; then `cfx3.sync` with the returned
  `syncedAt` → only changes.
- POST `cfx3.write {op:"node.create", …}` → `{ ok:true, uid }`; verify it appears in
  the next sync and re‑creating doesn't duplicate (clients update by uid).
