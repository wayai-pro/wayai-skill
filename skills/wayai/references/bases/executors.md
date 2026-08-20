# Executors

What an executor is, whether you need one, and the contract it must honor. Open this when a trigger or an external source has to act on the **outside world** — call a third-party API, present a client certificate, run logic the Data backend can't run itself.

An **executor** is a small, stateless HTTP service you deploy. The Data backend stays the system of record: it holds the records, brokers the credentials, and dispatches **signed** requests. Your executor verifies the request, does the real work, and writes the result back.

```
Agent → base (record + sign + dispatch) → executor (verify + act) → result back into the base
```

It sits on two directions, and both use the same signing scheme:

| Direction | Who calls whom | Surface |
|-----------|----------------|---------|
| **Outbound** — the base calls you | A **trigger** POSTs to your URL when records change; an **external source** proxies a record read/write through to your URL | `wayai triggers`, record-type sources |
| **Inbound** — you call the base | Your executor POSTs results into an **inbound webhook** | `wayai inbound-webhooks` |

The executor is invisible to the agent. Whether a record is native, source-backed, or executor-backed, the agent reads and writes it through the same `wayai records` / toolset surface — see [toolsets.md](toolsets.md).

## Table of Contents
- [Do you even need one?](#do-you-even-need-one)
- [The contract: verify → act → write back](#the-contract-verify--act--write-back)
- [The SDK and the wire](#the-sdk-and-the-wire)
- [Writing results back](#writing-results-back)
- [Where to run it](#where-to-run-it)
- [Certificate-bearing integrations (the credential-pull pattern)](#certificate-bearing-integrations-the-credential-pull-pattern)
- [Retries and dead-lettering](#retries-and-dead-lettering)
- [Local development](#local-development)

---

## Do you even need one?

Most integrations don't. Start here:

| Your upstream | What you build |
|---------------|----------------|
| A plain REST/JSON API the base can call directly | An **external source** with field mapping — **no executor** |
| A plain-HTTP upstream you want a native record type mirrored *out* to | An `external_write` trigger reusing that source's write path — **no executor** |
| A trigger that must run custom logic when records change | An executor |
| A source call that can't be a direct HTTP request the base makes itself — mutual TLS, SOAP or raw TCP, binary or per-tenant credentials, multi-call orchestration, heavy processing | An executor |

Sources, field mapping, and the trigger/inbound-webhook edges are covered in [integrations.md](integrations.md); the base entity model is in [README.md](README.md). Build an executor only when the table above says so — every executor is a service you now operate.

---

## The contract: verify → act → write back

Every dispatched request carries an **HMAC signature** over id + timestamp + method + path + body, a **timestamp** the receiver checks for freshness, and an **idempotency key** (falling back to the message id) used for dedupe — one scheme in both directions, so the same signer works for what you receive and what you send back. Your executor must:

- **Verify every request** — reject unsigned, invalid, or stale. Stale rejection is the replay defense; a signature check alone is not one.
- **Read the raw body bytes.** The signature covers exact bytes, so mount the handler *before* any JSON body parser. Re-serialized JSON will not verify.
- **Scope all work to the signed request.** Never trust an org id, base id, or tenant id read out of the request body — take identity from what was signed.
- **Dedupe on the idempotency key.** Delivery is at-least-once; a replayed delivery must return the first response, not act twice. The SDK dedupes with an in-memory store by default — if you run more than one instance, supply a **shared** store implementation, or dedupe is per-instance and the guarantee is fiction.
- **Stay stateless.** Credentials and records live in the base; the executor holds at most an ephemeral in-memory cache and no durable per-tenant state.
- **Return the normalized error envelope** `{ status, code, message, retriable }`. `retriable` → the base retries with backoff; a non-retriable `4xx` → the base stops and dead-letters. Getting this flag wrong is how a permanent upstream rejection turns into an infinite retry storm, or a transient blip silently drops a record.
- **Keep secrets and PII out of logs.** The dispatched body is tenant data.
- **Write the result back** — inline for a fast synchronous action, or via an inbound webhook for slow/async work (below).

The SDK gives you signature + freshness verification, idempotency dedupe, and the error envelope; you write one handler function.

---

## The SDK and the wire

**Never hand-roll verification.** Constant-time comparison, timestamp skew, signature-list rotation during secret rotation, and hashing the **exact raw body bytes** are each easy to get subtly wrong, and one mistake means anyone on the internet can forge a request into your executor. Use the published SDK:

```bash
npm i @wayai/executor-sdk
```

Zero dependencies, dual ESM/CJS, runs anywhere with Web Crypto (Node ≥24 per the package's declared engines, Cloudflare Workers, Deno, Bun).

```ts
import { createExecutor, toFetchHandler, ExecutorError } from '@wayai/executor-sdk'

const executor = createExecutor({
  secret: process.env.SIGNING_SECRET!,        // the source's or trigger's signing secret
  handler: async ({ body }) => {
    // Already verified: signature checked, timestamp fresh, retry deduped.
    // `body` is the RAW request string — the exact bytes the signature covers. Parse it yourself.
    const payload = JSON.parse(body) as { data?: unknown }
    if (!payload.data) {
      throw new ExecutorError({ code: 'MISSING_DATA', message: 'no data', retriable: false, status: 422 })
    }
    return { status: 200, body: JSON.stringify({ ok: true }) }
  },
})

export default { fetch: toFetchHandler(executor) }
```

The handler returns a `{ status, body, contentType? }` result — cached under the dedupe key and replayed verbatim on a duplicate delivery — or nothing at all, which answers `204`.

`toNodeHandler` is the Express/Node equivalent. Other exports: `verify` and `signRequest` (the primitives, if you are not using `createExecutor`), `fetchExecutorCredential` (the credential pull, below), `MemoryStore` plus the `IdempotencyStore` interface (swap in a shared store when you run more than one instance), `ExecutorError` / `toHttpStatus` / `jsonResponseBody` (the error envelope), and `DATA_HEADERS` (the header names below, as constants).

### Headers on every dispatch

| Header | Meaning |
|---|---|
| `X-Data-Signature` | `v1,<lowercase-hex HMAC-SHA256>` — a space-delimited list of `scheme,hex` entries, so a key rotation can emit old and new at once. Accept on **any** matching `v1` entry; skip unknown schemes. |
| `X-Data-Timestamp` | Unix **seconds**. Reject outside your freshness window — this, not the signature, is the replay defense. |
| `X-Data-Id` | The message id. **Bound into the signature**, so you must read it to rebuild the signed string. |
| `X-Data-Idempotency-Key` | On proxied writes only — the logical-operation identity, stable across retries of the same intent. |
| `X-Data-Delivery-Id` | On trigger webhooks only — a per-delivery id for tracing. **Not the dedupe key** (see below). |
| `X-Data-Credential-Url` / `-Token` / `-Names` | Present only when the source declares `executor_secrets` (see the credential-pull pattern below). |

Only `X-Data-Signature`, `-Timestamp` and `-Id` are on **every** dispatch; the rest are conditional as noted.

**Dedupe on `X-Data-Idempotency-Key`, falling back to `X-Data-Id`** — that pair, in that order, is what the SDK keys its store on. Do not key on `X-Data-Delivery-Id`: it is absent on proxied writes, so every proxied write would collide on one `undefined` key and replay the first response for all of them.

### What the signature covers

```
canonical = ["v1", id, timestamp, method, path, body].join("\n")
signature = HMAC-SHA256(secret, canonical)   → lowercase hex
```

`timestamp` is the integer seconds as a string, `method` the HTTP method, `path` the request `pathname + search`, and `body` the **exact raw body bytes** (`""` when there is none). Binding method and path is what stops a captured signature being replayed as a different request — so mount your handler *before* any JSON body parser; re-serialized JSON will not verify.

> **Do not confuse this with the API-channel webhook scheme.** WayAI's API-channel webhooks sign with `x-wayai-signature: v1=<hex>` over `"{timestamp}:{body}"` — different headers, a `=` instead of a `,`, and a different signed string. The two are separate surfaces; a verifier written for one will silently reject the other.

Timestamp skew is deliberately the **receiver's** call — the SDK's `verify` enforces it, the primitives leave it to you.

---

## Writing results back

A synchronous executor returns its result in the response body. For work that outlives the dispatch — a long upstream call, a queued job, a callback that lands later — POST the result into an **inbound webhook** on the base instead:

```
https://data.wayai.pro/inbound/<org>/<base>/<inbound_webhook_id>/ingest
```

Create the endpoint first. The shared secret is read from stdin, or prompted for when you omit the flag — it is never an argv element:

```bash
printf '%s' "$HOOK_SECRET" | wayai inbound-webhooks create --base <base> \
  --name executor-results --record-type-scope results --secret-stdin
```

Sign the write-back with the **same scheme in the opposite direction** — `signRequest` from the SDK is the inverse of its verify side, returning the `X-Data-Signature` / `X-Data-Timestamp` / `X-Data-Id` headers to attach, so one secret and one signer cover both edges. By default, writes arriving through an inbound webhook do **not** re-fire triggers, so an executor writing its own result back cannot loop.

Inbound webhooks can only be created or deleted in **preview** bases, then promoted. Payload translation, source bindings, and hydrating ("notification + fetch") webhooks are in [integrations.md](integrations.md).

---

## Where to run it

**Default to a serverless platform** — Cloudflare Workers, Vercel, Deno Deploy, AWS Lambda; whichever you already use, since the contract is platform-agnostic. Most executors are just authenticated API calls, and a serverless function is cheap, fast, scales to zero, and plugs straight into the SDK's Fetch handler.

**Step up to a long-running container or server** (Fly.io, Railway, Render, a VM — anything holding an open process, via the SDK's Node adapter) only when a serverless function genuinely can't do the job:

| Reason | Why serverless fails |
|--------|----------------------|
| **Mutual TLS / client certificates** (a bank or government API requiring an A1 cert) | Most serverless runtimes can't present a client cert on an outbound connection |
| **Long-running work** | Exceeds the function's CPU/wall-clock budget |
| **Raw TCP, SOAP, or other non-HTTP protocols** | No socket primitive |
| **Native dependencies** (heavy crypto, unhostable language runtimes) | No native module support |

These are exactly the cases the Data backend deliberately does not handle itself — which is why they live in the executor and nowhere else.

---

## Certificate-bearing integrations (the credential-pull pattern)

When an upstream requires a client certificate, store it as a **base credential** — held by the base itself, making it a first-class, rotatable, audited credential rather than a file baked into the executor image.

Create it on the base that dispatches to the upstream — `wayai bases credentials create`, the base's
Credentials tab, or the credential API directly. See [`toolsets.md`](toolsets.md#credentials-behind-a-bases-integrations)
for the full surface, including linking an organization credential instead of entering the value here.

```bash
# The credential value is PIPED, never an argv element: an argument leaks through /proc/<pid>/cmdline
# on a default host, shell history, and CI job logs. `openssl base64 -A` emits one unwrapped line
# (GNU `base64` wraps at 76 columns by default, which would put raw newlines inside the JSON string
# and get the request rejected as malformed); `jq -Rs` builds the body, trimming the trailing
# newline so the stored value is the certificate exactly. `expires_at` is metadata for your own
# renewal sweeps — nothing refuses a credential for being past it.
openssl base64 -A -in ./partner.p12 \
  | jq -Rs '{name: "partner-cert", value: rtrimstr("\n"),
             content_type: "application/x-pkcs12", expires_at: "2027-01-01T00:00:00Z"}' \
  | curl -X POST "https://data.wayai.pro/v1/$BASE/credentials" \
      -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' --data @-
```

A value is write-only on every verb: reads return metadata, never the stored credential.

Then declare it on the record type's source config (pushed with `wayai record-types upsert <id> --base <base> --name "<Name>" --sources @sources.json`) so the executor can **pull it at dispatch** — the right pattern for a binary or per-tenant credential too large to inject inline:

```json
{ "executor_secrets": [{ "name": "partner-cert", "secret_ref": "credential:partner-cert" }] }
```

On every proxied call the base advertises a **short-lived, single-use grant** to the executor, as the `X-Data-Credential-*` headers. Inside your **already-verified** handler, pull the credential by name with `fetchExecutorCredential`, passing the inbound request (it carries the grant):

```ts
import { fetchExecutorCredential } from '@wayai/executor-sdk'

const cert = await fetchExecutorCredential(request, 'partner-cert')
// → { name, contentType, value }   (the helper camelCases the wire's content_type)
```

- **Pull once per invocation and reuse** — the grant is single-use per credential name.
- The value comes back **verbatim**; decode it according to its recorded content type.
- The grant headers are **not signature-bound**. If you sit behind a TLS-terminating proxy, pin the allowed origin to `https://data.wayai.pro`.

A per-tenant certificate needs no templating: a reference already resolves against the credentials of the base serving the request, so the same `"secret_ref": "credential:partner-cert"` picks up each tenant base's own certificate. Placeholders are rejected in a `credential:` reference for that reason.

The certificate is never written to disk: the executor stays stateless, the pull is short-lived, single-use and audited, and rotating the credential on the base installs a new value **you supply** (it is never auto-generated) — the running executor picks it up on its next pull, with no redeploy.

Secrets are stripped from pulled config files and are never committed — see [config-as-code.md](config-as-code.md).

---

## Retries and dead-lettering

Delivery is **at-least-once**: failed deliveries are retried with backoff and dead-lettered after repeated failure. Inspect both directions:

```bash
wayai triggers deliveries --base <base> [--status pending|delivered|failed|dead] [--trigger-id <id>]
wayai inbound-webhooks deliveries --base <base> [--status pending|delivered|failed|dead] [--inbound-webhook <id>]
```

Because retries replay the same delivery, **idempotency dedupe is the only thing keeping at-least-once from becoming act-twice** — a duplicate charge, a duplicate email, a duplicate upstream record. Treat a `dead` delivery as a real incident: the base has stopped trying, and nothing else will resend it.

---

## Local development

The SDK exposes a **dev-bypass option** that skips signature verification while you develop locally (dedupe still applies when the headers are present). Gate it on an explicit environment check and **never enable it in production** — it disables the only thing standing between your executor and a forged request.

Point a trigger at your local or deployed executor. The signing secret is read from stdin or a masked prompt; there is no `--secret <value>` flag:

```bash
printf '%s' "$SIGNING_SECRET" | wayai triggers create --base <base> \
  --name notify-partner --events record.created,record.updated \
  --record-type-scope orders --url https://executor.example.com/hook --secret-stdin
```

Set that same value as your executor's **signing-secret environment variable** (name it whatever your deployment convention dictates). `--secret-prompt` is the interactive alternative. Set `WAYAI_BASE` to drop the repeated `--base`.

Triggers, like inbound webhooks, can only be created or deleted in **preview** bases — build and exercise the executor against a preview, then promote.

**Ship checklist:** verification on (bypass off) · raw-body handler mounted before any parser · shared dedupe store if replicated · error envelope with a deliberate `retriable` on every failure path · no secrets or PII in logs · certificate stored as a base credential, not baked into the image · `deliveries` checked after the first real dispatch.
