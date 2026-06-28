# Building a CF/x3 Client

> **⚠️ DRAFT.** Working draft; subject to change. Read the
> [Protocol specification](02-protocol.md) first — this guide is implementation
> advice, the spec is normative.

A **CF/x3 client** connects to one or more **sources**, mirrors their **context**
into a local store (tagged with origin, idempotently), and optionally **writes**
context back. This guide is language‑agnostic; pseudocode is illustrative.

## 1. Source registry

Hold one record per connected source. Keep API keys out of any synced/shared store.

```ts
interface Cfx3Source {
  id: string;            // stable local slug, e.g. "synaptic"
  name: string;
  url: string;           // CF/x3 endpoint, e.g. https://acme.example.com/cfx3
  apiKey: string;        // bearer credential (store securely)
  enabled: boolean;
  lastSyncAt?: string;   // the cursor — ISO timestamp from the last sync
  manifest?: Cfx3Manifest;       // cached manifest
  manifestRefreshedAt?: string;
}
```

## 2. Transport helper

One function issues a skill call and returns the result payload.

```
function cfx3Call(source, skill, payload):
    body = { jsonrpc:"2.0", id: uuid(), method:"tasks/send",
             params:{ message:{ role:"user", parts:[ { type:"data", data:{ skill, ...payload } } ] } } }
    res  = httpPost(source.url, headers={ "Content-Type":"application/json",
                                          "Authorization":"Bearer "+source.apiKey }, body)
    if res.status == 401: throw Cfx3Problem({ type:".../unauthenticated", status:401 })
    json = res.body
    if json.error:                                  # A2A JSON-RPC error; data = RFC 9457 problem
        throw Cfx3Problem(json.error.data or { title: json.error.message, status: 0 })
    task = json.result
    if task.status.state == "failed": throw task.status.message
    return firstDataPart(task.artifacts[0].parts).data     # the skill payload
```

`Cfx3Problem` wraps the RFC 9457 problem so callers can branch on `problem.type`
(e.g. `unauthenticated`, `insufficient-permission`) and show `problem.detail`.

Discovery (optional): `GET {url}/.well-known/agent.json` to read the card before
connecting; not required if you already know the endpoint.

CF/x3 is **sessionless**: there is no login. Send `Authorization: Bearer <apiKey>`
on **every** call; never assume the server remembers you between requests.

## 3. Permissions: who am I, what can I do

Permission is a **level 0–4 per context type** (`0 none · 1 read · 2 update ·
3 create · 4 delete`, cumulative). Learn the caller's levels via `cfx3.permissions`;
call it on connect and after a manifest refresh.

```
LEVEL = { none:0, read:1, update:2, create:3, delete:4 }

function refreshPermissions(source):
    p = cfx3Call(source, "cfx3.permissions", {})   # { subject, manageTypes, types: { <type>: 0..4 } }
    source.permissions = p; save(source)
    return p

levelOf(source, type)   = source.permissions?.types?.[type] ?? 0
canRead(source, type)   = levelOf(source, type) >= LEVEL.read    # 1
canUpdate(source, type) = levelOf(source, type) >= LEVEL.update  # 2
canCreate(source, type) = levelOf(source, type) >= LEVEL.create  # 3
canDelete(source, type) = levelOf(source, type) >= LEVEL.delete  # 4
```

The manifest also annotates each type with the caller's `level`, so a client that
only fetched the manifest already knows what it may do.

## 4. Manifest: fetch, cache, ensure types

```
function refreshManifest(source):
    m = cfx3Call(source, "cfx3.manifest", {})
    if m.cfx3_version != 1: warn("unsupported CF/x3 version", m.cfx3_version)
    ensureLocalTypes(m.context_types)      # create matching context types in your store
    source.manifest = m; source.manifestRefreshedAt = now(); save(source)
    return m
```

- Refresh on connect, then periodically (e.g. daily) or on demand.
- `ensureLocalTypes` maps each `Cfx3TypeDef` to your store's notion of a type
  (name, fields, `baseCategory` hint), creating it if absent. Do this **before**
  ingesting nodes so they validate.

## 5. Sync: read with a client‑owned cursor

```
function sync(source, opts):
    manifest = source.manifest or refreshManifest(source)
    since    = opts.full ? null : (source.lastSyncAt or null)
    cursor   = null
    repeat:
        res = cfx3Call(source, "cfx3.sync", { since, cursor })
        ingest(source, res.records, res.tombstones)
        cursor = res.nextCursor
    until cursor is null
    source.lastSyncAt = res.syncedAt; save(source)        # persist the new cursor
```

- **Full** sync: `since = null`. **Incremental:** send the stored `lastSyncAt`.
- Always persist the returned `syncedAt` as the next `since`. The client owns the
  cursor; never assume the server remembers it.
- Loop on `nextCursor` (same `since`) to drain pages, then store the final
  `syncedAt`.

## 6. Ingest: idempotent upsert keyed on (source, uid)

This is the heart of a client. Mirror each node into your store, **tagged with its
origin**, updating in place on re‑sync.

```
function ingest(source, records, tombstones):
    for rec in records:
        if rec.deleted: deleteLocalBySource(source.id, rec.uid); continue
        existing = findLocal(source.id, rec.uid)          # lookup by (source, uid)
        node = {
            ...(existing or {}),
            type:   rec.type,
            title:  rec.title,
            body:   rec.body or rec.summary or existing?.body,
            fields: rec.fields,
            links:  rec.links?.map(l => ({ label:l.label, target: resolveLocal(source.id, l.target) })),
            source:   source.id,          # PROVENANCE — which source
            sourceId: rec.uid,            # PROVENANCE — remote id (the idempotency key)
            updated:  rec.updated,
        }
        validateAndWrite(rec.type, node)  # apply your store's rules (types, etc.)
    for uid in tombstones:
        deleteLocalBySource(source.id, uid)
```

Rules:
1. **Primary key = `(source.id, uid)`.** Find an existing local node with the same
   source + remote uid; **update** it, else **create**. Re‑syncing never duplicates.
2. **Record provenance** (`source`, `sourceId`) on every node, so you can re‑sync,
   diff, and cleanly remove a source's data later.
3. **Links:** the `target` is a remote `uid`. Resolve it to a local reference once
   both nodes exist; if the target isn't ingested yet, keep the remote uid and
   resolve on a later pass.
4. **Deletes:** remove local nodes for tombstoned uids or `deleted:true` records.
5. Apply your own store's validation/normalization on write.

## 7. Write‑back (optional)

To push a change to the source:

```
# create
cfx3Call(source, "cfx3.write", { op:"node.create",
    node:{ type:"contact", title:"Sam Lee", fields:{ email:"sam@acme.com" },
           links:[ { label:"works-at", target:"crm/company/CO123" } ] } })
# update an existing mirrored node — use its stored remote uid (sourceId)
cfx3Call(source, "cfx3.write", { op:"node.update",
    node:{ uid: node.sourceId, fields:{ phone:"+1…" } } })
# delete
cfx3Call(source, "cfx3.write", { op:"node.delete", uid: node.sourceId })
```

- Check the caller's **level** for the type before offering an op (`canUpdate` ≥2,
  `canCreate` ≥3, `canDelete` ≥4).
- For an existing mirrored node, **update by its `sourceId`** so the source mutates
  in place (no duplicate).
- A rejected write comes back as an RFC 9457 problem (e.g. 403
  `insufficient-permission`); surface its `detail` (see §9).

## 8. Scheduling & refresh

- Run `sync` on demand (a "Sync now" affordance) and/or on a schedule (e.g. hourly).
- Refresh the manifest less often than you sync.
- A reasonable cron loop: for each enabled source → refresh manifest if stale →
  incremental sync.

## 9. Errors & resilience

- Errors arrive as A2A **JSON‑RPC error** responses whose `error.data` is an
  **RFC 9457** problem (spec §7). Branch on `problem.type`:
  - `unauthenticated` (401) → the token is bad/expired; prompt to reconnect.
  - `insufficient-permission` (403) → hide/disable the affordance; surface `detail`.
  - `not-found` / `unprocessable` → surface `detail`; for `unprocessable`, show
    `problem.errors` (per‑field).
- Log and continue to the next source; don't let one bad source block others.
- Time‑box network calls so a slow/offline source can't hang your sync loop.
- Treat a version mismatch (`cfx3_version`) as a soft failure: skip or degrade.

## 10. Minimal client checklist

- [ ] Source registry with `url`, `apiKey`, `lastSyncAt` (key stored securely).
- [ ] `cfx3Call` transport (POST `tasks/send`, extract first data artifact).
- [ ] Manifest fetch + cache + `ensureLocalTypes`.
- [ ] `sync`: full/incremental, paging, persist `syncedAt`.
- [ ] `ingest`: upsert keyed on `(source, uid)`, record provenance, resolve links,
      apply tombstones.
- [ ] (optional) write‑back via `cfx3.write`, updating existing nodes by `sourceId`.
