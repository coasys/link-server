# @coasys/adam-link-server

A self-hostable link language server for [AD4M](https://github.com/coasys/ad4m). Communities run this on their own hardware; AD4M agents authenticate with their DID and sync link data through it. Think Matrix homeserver, but purpose-built for AD4M link sync instead of chat.

**Companion repo:** [`adam-server-link-language`](https://github.com/coasys/adam-server-link-language) — the AD4M link language that talks to this server.

## Quickstart

```bash
npx @coasys/adam-link-server --port 3456 --data ./my-data
```

Or with Docker:

```bash
docker compose up
```

The server generates its own ed25519 identity keypair on first run (`<data-dir>/data.sqlite`, `server_identity` table) and creates rooms on demand — there's no separate provisioning step.

## How it works

- **Rooms** are independent link-sync spaces, identified by an opaque `roomId` the client chooses. The first agent to authenticate against a room becomes its admin.
- **Auth** is DID challenge-response: an agent proves control of its `did:key` ed25519 key by signing a server-issued nonce, and receives a JWT scoped to `(did, roomId)`.
- **ACL** gates every room endpoint. Only the admin can add/remove DIDs.
- **Links** are stored as an append-only diff log (`PerspectiveDiff` = additions/removals of signed `LinkExpression`s) plus a derived active-set table, so the room's state is always `replay(diffs)`. The **revision** is a content hash of the active set's link hashes — order-independent, so two servers with the same active links converge to the same revision regardless of how they got there.
- **WebSocket** push delivers committed diffs and telepresence events in real time.
- **Federation** forwards committed diffs to peer servers (server-to-server, authenticated by the sending server's own ed25519 signature) and offers pull-based reconciliation to catch up on anything missed.
- **E2E encryption** is opt-in per room: link `data` becomes an opaque ciphertext blob the server cannot read, while `author`/`timestamp`/`proof` stay visible so the server can still enforce ACL and OR-Set merge.

See [`AGENTS.md`](./AGENTS.md) for architecture, file layout, and implementation decisions made where the spec was ambiguous.

## API

All endpoints except `/rooms/:roomId/auth`, `/server/identity`, and the federation transport (`/federate`, `/reconcile`) require `Authorization: Bearer <jwt>`.

```
POST /rooms/:roomId/auth      { did } -> { challenge }
                               { did, challenge, signature } -> { token, expiresAt }
POST /rooms/:roomId/commit    { additions: LinkExpression[], removals: LinkExpression[] } -> { sequence, revision }
GET  /rooms/:roomId/sync      ?since=<sequence> -> { diffs: PerspectiveDiff[], revision, sequence }
GET  /rooms/:roomId/render    -> { links: LinkExpression[], revision }
GET  /rooms/:roomId/revision  -> { revision, sequence }
GET  /rooms/:roomId/peers     -> { peers: string[] }               (currently online agents)
POST /rooms/:roomId/acl       { action: "add"|"remove", did } (admin only)
GET  /rooms/:roomId/acl       -> { admin, members: string[] }
POST /rooms/:roomId/federation { action: "add"|"remove", peerUrl } (admin only)
GET  /rooms/:roomId/federation -> { peers: string[] }
POST /rooms/:roomId/federate   (peer servers only, signature-authenticated)
POST /rooms/:roomId/reconcile  (peer servers only, signature-authenticated)
GET  /rooms/:roomId/keys       -> { encryptedKey, version } | 404
POST /rooms/:roomId/keys/rotate (admin only) -> { version, recipients }
GET  /server/identity          -> { publicKey }
GET  /rooms/:roomId/ws?token=<jwt>  (WebSocket upgrade)
```

### WebSocket messages

Server -> client: `diff`, `telepresence-signal`, `telepresence-broadcast`, `online-agents`, `peer-joined`, `peer-left`.
Client -> server: `telepresence-signal { toDid, payload }`, `telepresence-broadcast { payload }`, `set-online-status { status }`.

### Rate limits

100 req/min per IP on `/auth`, 300 req/min per JWT on room endpoints, 60 req/min per JWT on `/commit` specifically (stacked on top of the general room limit). Sliding window, in-memory. `429` responses carry `Retry-After` in seconds.

## Development

```bash
npm install       # NODE_ENV must not be "production" or devDependencies won't install
npm test          # node's built-in test runner, tests/*.test.ts
npm run build     # tsc -> dist/
npm run dev       # tsx src/index.ts, no build step
```

Tests boot a real server per test (random port, temp SQLite file) and drive it over real HTTP/WebSocket — there are no mocks of the server itself.
