# hasura_build

Komodo stack running [Hasura GraphQL Engine](https://hasura.io/) v2
(Community Edition) against an existing Postgres.

---

## Storage

Hasura has **no embedded storage**. It keeps its metadata — tracked tables,
relationships, permissions, remote schemas, event triggers — in Postgres.

This stack deliberately ships **no database of its own**. Point
`HASURA_DB_*` at a Postgres instance that is already backed up, and the
metadata is covered by that instance's existing backup job. Running a
second, unbackedup Postgres beside Hasura is the easy mistake here: losing it
loses every permission rule and relationship you ever configured.

Create the role and database once:

```sql
CREATE ROLE hasura LOGIN PASSWORD '...';
CREATE DATABASE hasura OWNER hasura;
```

---

## Security

`HASURA_ADMIN_SECRET` is **required** — the compose file refuses to start
without it. This is not a formality:

> With no admin secret, Hasura's GraphQL API and console are completely
> unauthenticated. Anyone who can reach the port can read and write every
> tracked table, and run arbitrary SQL through the console.

Other defaults chosen with that in mind:

| Setting | Default | Why |
| --- | --- | --- |
| `HASURA_GRAPHQL_DEV_MODE` | `false` | dev mode returns internal error details to clients |
| `HASURA_GRAPHQL_UNAUTHORIZED_ROLE` | *(not set)* | unauthenticated requests are rejected outright |
| `HASURA_BIND` | `127.0.0.1` | nothing is exposed off-host unless you ask |
| `HASURA_GRAPHQL_CORS_DOMAIN` | `*` | **widen-open; narrow this to your origins** |

### Anonymous access

`HASURA_GRAPHQL_UNAUTHORIZED_ROLE` names the role given to requests that carry
no authentication. Leave it unset and such requests are rejected outright; set
it — conventionally to `anonymous` — and they are evaluated as that role.

Setting it is safe on its own: the role grants nothing until you define
permissions for it in the console. It only widens access once you do.

It must be **absent rather than empty** — Hasura rejects an empty string with
`empty string not allowed` and crash-loops. That is why `compose.yaml` uses
list-form `environment` with a bare key, which Compose passes through only
when the variable is actually set.

If you expose the console to the internet, put an authenticating proxy
(Cloudflare Access, or similar) in front of it. The admin secret is a single
shared password with no MFA and no lockout.

---

## Setup

```bash
docker network create cloudflared 2>/dev/null || true
cp .env.example .env    # then edit it
docker compose up -d
```

Check it came up:

```bash
curl -fsS http://127.0.0.1:8080/healthz && echo OK
```

The console is at `/console`, and asks for the admin secret.

---

## Behind a reverse proxy or tunnel

The container joins the external `cloudflared` network, so anything else on
that network reaches it as `http://hasura:8080` — no published port needed.
Point a proxy host or tunnel public hostname there.

Hasura uses **websockets** for GraphQL subscriptions. Enable websocket
support on the proxy or subscriptions will fail while queries appear fine.

---

## Files

| File | Purpose |
| --- | --- |
| `compose.yaml` | the stack definition |
| `.env.example` | template for `.env` |
