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
- For each type, list the **fields** you'll expose (key + type), and mark types the
  caller can't mutate as `readonly`.
- Use **`links`** for relationships, with `target` set to another node's `uid`
  (e.g. a contact links `works-at` → `crm/company/CO123`).

Example mapping (a CRM):

| Your table | Context type | uid | fields |
|------------|-------------|-----|--------|
| companies | `company` | `crm/company/{id}` | domain, industry, city, … |
| contacts | `contact` | `crm/contact/{id}` | email, role, … (+ link to company) |
| ledger accounts | `account` (readonly) | `acc/account/{id}` | code, type, … |

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

```
function handlePost(req):
    body = parseJson(req)                       # else JSON-RPC -32700
    if body.method != "tasks/send": return rpcError(-32601)
    data  = firstDataPart(body.params.message.parts) or {}
    skill = data.skill
    auth  = authenticate(req)                    # 401 if required and missing/invalid
    perms = loadPermissions(auth)

    switch skill:
      "cfx3.manifest": return rpcResult(body.id, "cfx3.manifest", buildManifest(perms))
      "cfx3.sync":     return rpcResult(body.id, "cfx3.sync",     runSync(perms, data.since, data.types, data.cursor))
      "cfx3.write":    return rpcResult(body.id, "cfx3.write",    runWrite(perms, data))
      default:         return rpcError(body.id, -32602, "Unknown CF/x3 skill")
```

Where `rpcResult` wraps the payload in a completed A2A task artifact:

```
rpcResult(id, skill, payload) =
  { jsonrpc:"2.0", id, result:{ id: newId(), status:{state:"completed"},
      artifacts:[ { name: skill, parts:[ { type:"data", data: payload } ] } ] } }
```

## 4. `cfx3.manifest`

Return the context types **filtered to what the caller may read**, plus
capabilities derived from the caller's permissions:

```
buildManifest(perms) =
  { cfx3_version: 1,
    source: "<your-source-id>",
    name:   "<display name>",
    context_types: TYPES.filter(t => perms.canRead(t)).map(stripInternalFields),
    capabilities: {
      writeNodes:  perms.canWriteNodes,
      deleteNodes: perms.canDeleteNodes,
      manageTypes: false              # true only if types are user-definable
    },
    auth: { schemes: ["bearer"] } }
```

Mark types the caller cannot mutate with `readonly: true`.

## 5. `cfx3.sync`

Translate each record into a `Cfx3Node`. Honor the cursor and type filter.

```
runSync(perms, since, types, cursor):
    records = []
    for type in TYPES where perms.canRead(type) and (types is empty or type in types):
        rows = query(type, where: since ? "updated_at > :since" : "true",
                     page: cursor, limit: PAGE)
        for row in rows:
            records.push(toNode(type, row))     # uid, type, title, fields, links, updated
    tombstones = since ? deletedUidsSince(since, perms) : []   # [] if you don't track deletes
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
runWrite(perms, data):
    op = data.op
    if op starts with "type." and not perms.canManageTypes:
        return { ok:false, message:"type management not supported" }

    if op == "node.delete":
        if not perms.canDeleteNodes: return { ok:false, message:"forbidden" }
        (type, id) = parseUid(data.uid)
        deleteRecord(type, id); return { ok:true, uid:data.uid }

    if op in ("node.create","node.update"):
        if not perms.canWriteNodes: return { ok:false, message:"forbidden" }
        node = data.node
        type = node.type or parseUid(node.uid).type
        if not writable(type): return { ok:false, message:"type not writable" }
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
- Reject writes to `readonly` types.

## 7. Permissions

- Authenticate the bearer key to a principal.
- Derive **read** scope (which types) and **write/delete** capability from the
  principal's roles.
- Filter the manifest accordingly and gate every `cfx3.sync`/`cfx3.write` call.

A single API credential commonly maps to a user/role set; the server returns only
what that role may see and do.

## 8. Errors

- Malformed / unsupported envelope → JSON‑RPC errors (`-32700/-32601/-32602`).
- Skill execution failure → `-32000` with a message (or, for writes, `ok:false`).
- Unauthenticated → 401.

## 9. Keep actions out

If your system also has domain actions (state transitions, sends, charges), expose
them via a **separate MCP server**, not CF/x3. CF/x3 stays context‑only so clients
can mirror you safely. See [Why context federation](01-why-context-federation.md).

## 10. Test your server

- `curl {base}/.well-known/agent.json` → card with 3 skills.
- POST `cfx3.manifest` with a bearer key → context types + capabilities.
- POST `cfx3.sync {since:null}` → records; then `cfx3.sync` with the returned
  `syncedAt` → only changes.
- POST `cfx3.write {op:"node.create", …}` → `{ ok:true, uid }`; verify it appears in
  the next sync and re‑creating doesn't duplicate (clients update by uid).
