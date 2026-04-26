# Permissions and ACL

How identity, authentication, and access control work across gosh.memory,
gosh.agent, and gosh.cli. Read this if you are building a multi-agent
swarm, exposing memory to a third party, or trying to understand who is
allowed to read or write a given fact.

---

## Why this exists

gosh.memory holds the long-term state of every agent and human in a
deployment. Every fact has an owner, a scope, and an audience. The
permission model is what lets two agents share a swarm without leaking
private memos, lets the operator inspect anything as admin, and lets
external clients (Claude Code, Codex, Gemini, agents on other machines)
talk to memory through MCP without a shared password.

There are three concepts you have to keep separate:

| Concept | What it is | Where it lives |
|---------|------------|---------------|
| **Principal** | An identity (user, agent, or service) | `principals` table in `authority.db` |
| **Token** | A bearer credential that authenticates a principal on a request | `tokens` table in `authority.db` |
| **Scope / membership** | What a fact is visible to (private, swarm, system) | `scope` field on each fact + `memberships` table |

`agent_id` and `swarm_id` on memory calls are **selectors** (which
agent's facts, which swarm), not authentication. The token determines
who you are. The selectors determine what you are looking at.

---

## Principals

A principal is `{kind}:{id}` where:

- `kind` ∈ `user` | `agent` | `service`
- `id` matches `^[A-Za-z0-9._:@/-]+$`

Examples:

```
user:alice@example.com
agent:coder-a
service:gosh-memory
```

Principals live in `authority.db` and are managed via the auth tools:

| MCP tool | gosh.cli command | Purpose |
|----------|------------------|---------|
| `principal_create` | `gosh memory auth principal create <ID> --kind <KIND>` | Create a principal |
| `principal_get` | `gosh memory auth principal get [ID]` | Read a principal record |
| `principal_disable` | `gosh memory auth principal disable <ID>` | Disable a principal (tokens stop resolving) |

A principal does not by itself grant access. It needs at least one token
(to authenticate calls) and, for swarm-scoped data, at least one
membership.

---

## Tokens

A token is a bearer string sent to memory either as the `token` field on
an MCP tool call or as the `Authorization: Bearer <token>` header on
HTTP. Resolution returns the principal, role, and current memberships
for that token.

### Token kinds

| Kind | Caller role | Use |
|------|-------------|-----|
| `bootstrap` | admin | One-time pairing token used by `auth_bootstrap_admin` to mint the first admin |
| `admin` | admin | Long-lived operator token; bypasses fact ACL filtering |
| `user` | user | Regular human user token |
| `agent` | user | Agent-bound token; agents present this on every memory call |
| `join` | n/a | TLS join token containing memory URL + transport token + cert pin, decoded by `gosh-agent --join` to bootstrap a remote agent |

The bootstrap token is set in the `GOSH_MEMORY_ADMIN_TOKEN` environment
variable when memory starts. After bootstrap completes, that token is
sealed and cannot be used again.

### Token lifecycle

| MCP tool | gosh.cli command | Purpose |
|----------|------------------|---------|
| `auth_bootstrap_admin` | (called automatically by `gosh memory start` on first run) | Mint the first admin token from the bootstrap token |
| `auth_token_issue` | `gosh memory auth token issue <PRINCIPAL_ID> --kind <KIND>` | Mint a new token |
| `auth_token_revoke` | `gosh memory auth token revoke <TOKEN_ID>` | Revoke a token |
| `auth_token_list` | `gosh memory auth token list [--principal-id <ID>]` | List tokens for a principal |

Tokens carry `issued_at`, optional `expires_at`, and `revoked_at`
fields. Resolution fails for revoked, expired, or disabled-principal
tokens with one of these error codes:

| Code | Meaning |
|------|---------|
| `AUTH_REQUIRED` | No token presented |
| `INVALID_TOKEN` | Token does not exist |
| `AUTH_DISABLED` | Owning principal is disabled |
| `AUTH_REVOKED` | Token revoked |
| `AUTH_EXPIRED` | Token past `expires_at` |
| `FORBIDDEN` | Token resolved but caller lacks permission for this action |

### Where tokens are stored on the client side

gosh.cli writes tokens into the OS keychain (or, in `--test-mode`, a
file-backed keychain) using two service entries:

```
gosh / memory/<instance>   →  encryption_key, bootstrap_token, server_token,
                              admin_token, agent_token
gosh / agent/<name>        →  principal_token, join_token, secret_key (X25519)
```

`gosh agent bootstrap export|show|rotate` operate on the agent entry.
`gosh memory auth provision-cli` writes the `agent_token` for use by
`gosh memory data ...` commands.

---

## The two-layer transport

Memory exposes HTTP at `http(s)://HOST:PORT`. Two distinct credentials
gate two different layers:

1. **Server token** (`x-server-token` header, env `GOSH_MEMORY_TOKEN`)
   — perimeter auth, sufficient for `/health` and the MCP handshake
   only. It does not authorize any data plane operation.
2. **Principal token** (the `token` field on MCP tool calls, or
   `Authorization: Bearer <token>`) — authenticates the caller's
   principal and authorizes data operations.

A direct API integration must present both: `x-server-token` to get
through the perimeter, and a principal token to do anything once
inside.

---

## Scopes

Every stored fact has an explicit `scope` field. Scope is **not derived
from context** — every `memory_store`, `memory_write`, and `memory_ingest_*`
call must pass `scope` explicitly or it errors with "scope must be
provided explicitly".

| Scope | Visible to |
|-------|-----------|
| `agent-private` | Only the owning agent (and admins) |
| `swarm-shared` | All members of the named swarm (`swarm_id`) |
| `system-wide` | Every authenticated principal |

The CLI default is `agent-private` for `gosh memory data store` and
`swarm-shared` is required for any data the agent wants its swarm
peers to see.

`agent-private` is reserved for the namespace owner. A non-owner
principal writing into a swarm-bound namespace must use
`scope=swarm-shared` or the write is rejected.

### ACL fields on each fact

Beyond `scope`, the fact carries explicit ACL fields, derived at write
time from the caller and the chosen scope:

```json
{
  "owner_id": "agent:coder-a",
  "scope": "swarm-shared",
  "read":  ["swarm:alpha"],
  "write": ["swarm:alpha"],
  "_derived_read":  [...],
  "_derived_write": [...]
}
```

| Field | Meaning |
|-------|---------|
| `owner_id` | Principal that wrote the fact. Always has full access. |
| `read` | Principals or `swarm:X` references granted read |
| `write` | Principals granted modify (edit/retract) rights |
| `_derived_*` | System-expanded ACL — e.g., `swarm:alpha` expanded to its current member list |

---

## Memberships and swarms

A swarm is a named group of principals. Membership grants those
principals read/write access to facts whose ACL references `swarm:<id>`.

| MCP tool | gosh.cli command | Purpose |
|----------|------------------|---------|
| `swarm_create` | `gosh memory auth swarm create <NAME> --owner <PRINCIPAL_ID>` | Create a swarm with an owner |
| `swarm_get` | `gosh memory auth swarm get <ID>` | Read a swarm record |
| `swarm_list` | `gosh memory auth swarm list` | List all swarms |
| `membership_grant` | `gosh memory auth membership grant <PRINCIPAL_ID> --swarm <SWARM> [--role member\|manager\|owner]` | Add a principal to a swarm |
| `membership_revoke` | `gosh memory auth membership revoke <PRINCIPAL_ID> --swarm <SWARM>` | Remove a principal |
| `membership_list` | `gosh memory auth membership list [--swarm <SWARM>]` | List memberships |
| `membership_register` | (no CLI; called by agents) | Self-register the calling principal in a group, for the lifetime of the token |
| `membership_unregister` | (no CLI) | Remove a self-registration |

Memberships have an optional `expires_at`. Expired memberships are
auto-revoked on the next access.

### Per-call selector vs. persistent membership

There are two ways for an agent to act in a swarm:

- **Per-call selector**: pass `swarm_id="alpha"` on each MCP call. The
  call is scoped to that swarm. The principal does not need a
  persistent membership record; it gets a synthetic `swarm:alpha`
  membership for the duration of the call.
- **Persistent membership**: an admin runs `membership_grant` once.
  The principal is then a true member; calls without `swarm_id` still
  see facts that match the principal's memberships.

Persistent membership is required for watch mode, courier
subscriptions, and anything where the agent is not threading
`swarm_id` through every request. Per-call selector is fine for
short-lived scripts or operator commands.

### Roles inside a swarm

| Role | Capabilities |
|------|--------------|
| `owner` | Full control of the swarm; can grant/revoke any role, delete the swarm |
| `manager` | Grant/revoke `member` roles; cannot remove the owner |
| `member` | Read/write swarm-shared facts; cannot manage membership |

The CLI defaults to `member` if `--role` is omitted on
`gosh memory auth membership grant`.

---

## Instance-level ACL

When you first write into a memory namespace (key), memory creates an
`_instance_config` entry that records the instance owner and instance-
level read/write ACL. This is a second filter that runs **before** the
fact-level ACL: a principal that is not allowed at the instance level
sees nothing in that namespace, regardless of fact-level ACL.

In practice this matters when:

- Multiple swarms share one memory server but must not see each other's
  namespaces.
- An admin wants to lock a namespace to a specific owner principal so
  that bootstrap or recovery does not let other admins read it.

The instance config is auto-derived for new namespaces from the first
writer's principal. To pre-create one, use:

```bash
gosh memory init --key <KEY> [--owner-id <PRINCIPAL>]
```

---

## Bootstrap flow

A fresh memory has no principals. Bootstrapping the first admin works
like this:

1. Operator generates a random `bootstrap_token` (32 bytes, base64url)
   when running `gosh memory setup local` and stores it in the OS
   keychain.
2. `gosh memory start` exports it as `GOSH_MEMORY_ADMIN_TOKEN` to the
   memory process.
3. On first start, gosh.cli detects no `admin_token` is in the keychain
   yet, calls `auth_bootstrap_admin`, receives a long-lived admin
   token, and stores it in the keychain.
4. The bootstrap token is then sealed in `bootstrap_state` — using it
   again returns `BOOTSTRAP_ALREADY_USED`.

From that point on, the admin token is used for all auth/principal/
swarm/membership/secret operations from the operator side.

---

## Secrets

API keys and other credentials that the memory pipeline needs at
runtime (extraction model keys, embedding keys, etc.) are stored inside
memory itself, encrypted, and referenced by name from `profile_configs`.

| MCP tool | gosh.cli command |
|----------|------------------|
| `memory_store_secret` | `gosh memory secret set <NAME> <VALUE>` |
| `memory_store_secret` | `gosh memory secret set-from-env <ENV_VAR> --name <NAME>` |
| `memory_list_secrets` | `gosh memory secret list` |
| `memory_delete_secret` | `gosh memory secret delete <NAME>` |

Secrets have a `scope` like facts: `system-wide`, `swarm-shared`, or
`agent-private`. They are **write-only** from the API surface — there
is no `secret_get` tool. The only way memory's runtime sees a secret
value is through internal lookup at execution time.

In `profile_configs`, the `secret_ref` field names a secret:

```json
{
  "extraction": {
    "model": "anthropic/claude-sonnet-4-6",
    "secret_ref": {"name": "anthropic", "scope": "system-wide"},
    "pricing": {"input_per_1k": 0.003, "output_per_1k": 0.015}
  }
}
```

`secret_ref.name` is just a label, not a routing key. The provider is
chosen by the prefix on `model` (`anthropic/`, `qwen/`, `meta-llama/`,
`google/`; bare names go to OpenAI).

### X25519-sealed delivery to agents

When an agent needs a secret (e.g., to call its inference model), memory
encrypts the secret to the agent's X25519 public key and returns the
ciphertext in the plan. Only the agent — which holds the matching
private key in its own keychain — can decrypt it. The memory server
never logs the plaintext.

The construction is **X25519 + HKDF-SHA256 + AES-256-GCM**, identified
by the algorithm tag `x25519-hkdf-sha256-aes256gcm-v1` in
`src/gosh_secrets.py`. It is not libsodium `crypto_box_seal` / NaCl
`SealedBox` — the term "sealed envelope" is descriptive, not a
reference to the libsodium primitive of the same name.

This is why `gosh agent create` generates an X25519 keypair and
registers the public key with memory: it makes secret delivery possible
without ever putting the private key in a fact.

---

## Encryption at rest

If `GOSH_MEMORY_ENCRYPTION_KEY` is set (32 bytes, hex-encoded) when
memory starts, `authority.db` (principals, tokens, swarms, memberships,
bootstrap state) is encrypted at rest using AES-GCM. `gosh memory setup
local` generates this key automatically and stores it in the OS keychain
alongside the bootstrap token; the start command exports it back to the
memory process.

Fact data in `memory_<key>.db` is **not** encrypted at rest — protect
the data directory at the filesystem layer.

---

## What admin can do that other principals cannot

Admins (`caller_role="admin"`) get these privileges:

- Bypass fact-level ACL filtering on read (still subject to instance ACL)
- Mint, revoke, and list any principal's tokens
- Create swarms and grant memberships to any principal
- Create, get, disable any principal
- See `principal_disable` and revoked tokens in lists
- Run `memory_admin_backfill_original_raw_sources`

Admin is a **role on the token**, not a principal kind. A `service:`
principal can hold an admin token; an `agent:` principal can hold an
admin token; whether the role on the resolved token is `admin` is what
matters.

---

## Provisioning the CLI as an agent

`gosh memory data store|recall|ask|...` need an `agent_token` (not the
admin token) because the data plane is designed for agent identities,
not operator identities. The CLI provides a one-shot bootstrap:

```bash
gosh memory auth provision-cli
```

This is idempotent. It:

1. Creates principal `agent:cli-<your-username>` (skips if exists)
2. Creates swarm `cli` owned by that principal (skips if exists)
3. Grants membership of the principal in the swarm (skips if exists)
4. Issues an `agent`-kind token for the principal
5. Saves the token to the keychain under `gosh / memory/<instance>`

After provisioning, all `gosh memory data ...` commands work without
further auth setup.

---

## Common errors and what they mean

| Error | What to do |
|-------|-----------|
| `data commands ... require an agent token` | Run `gosh memory auth provision-cli`. |
| `scope must be provided explicitly` | Pass `--scope agent-private\|swarm-shared\|system-wide` on the call. |
| `BOOTSTRAP_ALREADY_USED` | The bootstrap token has been consumed. Use the saved admin token from the keychain (`gosh agent bootstrap show` or look at `gosh memory auth principal get`). |
| `AUTH_REQUIRED` | Token not presented. CLI commands handle this automatically; for direct MCP calls, pass `token=...` or `Authorization: Bearer ...`. |
| `FORBIDDEN` writing to `agent-private` in a swarm namespace | You are not the namespace owner. Use `scope=swarm-shared`. |
| Agent can't see swarm-shared facts | Either pass `swarm_id` on every call, or grant a persistent membership: `gosh memory auth membership grant agent:<name> --swarm <SWARM>`. |

---

## Related docs

- [SETUP.md](SETUP.md) — install and bootstrap commands in context
- [MEMORY-SYSTEM.md](MEMORY-SYSTEM.md) — how facts, retrieval, and ACL filtering interact
- [GOSH-SWARM-USAGE-GUIDE.md](GOSH-SWARM-USAGE-GUIDE.md) — practical multi-agent example
- [GOSH-SWARM-COORDINATION.md](GOSH-SWARM-COORDINATION.md) — coordination protocol that uses these primitives
