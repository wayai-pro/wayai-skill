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
- [What this file does NOT carry](#what-this-file-does-not-carry)
- [Do you even need one?](#do-you-even-need-one)
- [The contract: verify → act → write back](#the-contract-verify--act--write-back)
- [Writing results back](#writing-results-back)
- [Where to run it](#where-to-run-it)
- [Certificate-bearing integrations (the vault pattern)](#certificate-bearing-integrations-the-vault-pattern)
- [Retries and dead-lettering](#retries-and-dead-lettering)
- [Local development](#local-development)

---

## What this file does NOT carry

> **You cannot write the verify step from this file.** It deliberately does not carry the request-signing
> contract's identifiers — the verification SDK's package name and exports, the signature / timestamp /
> idempotency **header names**, or the canonical-string construction, digest and encoding the signature
> is computed over. Those are published with the Data backend and are the part of this surface still
> settling as the Data layer folds into the platform, so a skill that pinned a spelling would hand out
> a stale one.
>
> **So scope your expectations of this file:** it tells you *whether* you need an executor, what the
> service must guarantee, where to run it, how credentials reach it, and how failures are retried. For
> the bytes on the wire, get the current SDK and header spellings from the Data backend's own
> executor documentation before you write the handler.

Verification is the part worth not improvising. Constant-time comparison, timestamp skew, signature-list rotation during secret rotation, and hashing the **exact raw body bytes** are each easy to get subtly wrong, and one mistake means anyone on the internet can forge a request into your executor — which is why the published SDK exists and why this file will not half-specify it.

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

Every dispatched request carries an **HMAC signature** over id + timestamp + method + path + body, a **timestamp** the receiver checks for freshness, and a **delivery id** used for dedupe — one scheme in both directions, so the same signer works for what you receive and what you send back. Your executor must:

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

Sign the write-back with the **same scheme in the opposite direction** — the SDK's signing helper is the inverse of its verify side, so one secret and one signer cover both edges. By default, writes arriving through an inbound webhook do **not** re-fire triggers, so an executor writing its own result back cannot loop.

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

## Certificate-bearing integrations (the vault pattern)

When an upstream requires a client certificate, store it in the **vault** — org-scoped, so one credential serves every base in the org — making it a first-class, rotatable, audited credential rather than a file baked into the executor image.

```bash
# --file reads the .p12/.pem and base64-encodes it; --content-type records the format.
# (--value-stdin / --value-prompt are the alternatives for text values. There is no
#  --value <v> flag: an argv element leaks through /proc, shell history, and CI logs.)
wayai bases secrets create --name partner-cert --file ./partner.p12 \
  --content-type application/x-pkcs12 --expires-at 2027-01-01T00:00:00Z
```

Then declare it on the record type's source config (pushed with `wayai record-types upsert <id> --base <base> --name "<Name>" --sources @sources.json`) so the executor can **pull it at dispatch** — the right pattern for a binary or per-tenant credential too large to inject inline:

```json
{ "executor_secrets": [{ "name": "partner-cert", "secret_ref": "vault:partner-cert" }] }
```

On every proxied call the base advertises a **short-lived, single-use grant** to the executor. Inside your **already-verified** handler, pull the credential by name with the SDK's credential-pull helper, passing the inbound request (it carries the grant):

- **Pull once per invocation and reuse** — the grant is single-use per credential name.
- The value comes back **verbatim**; decode it according to its recorded content type.
- The grant headers are **not signature-bound**. If you sit behind a TLS-terminating proxy, pin the allowed origin to `https://data.wayai.pro`.

For a per-tenant certificate, template the reference — `"secret_ref": "vault:partner-cert-{{auth.base_id}}"` resolves each calling base's own credential. Only `{{auth.org_id}}` and `{{auth.base_id}}` are allowed in that position.

The certificate never leaves the vault for disk: the executor stays stateless, the pull is scoped and audited, and rotation is central — `wayai bases secrets rotate <id>` installs a new value **you supply** (it is never auto-generated), and `wayai bases secrets list --expiring --days 30` surfaces what is about to lapse before an upstream starts rejecting handshakes.

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

**Ship checklist:** verification on (bypass off) · raw-body handler mounted before any parser · shared dedupe store if replicated · error envelope with a deliberate `retriable` on every failure path · no secrets or PII in logs · certificate in the vault, not the image · `deliveries` checked after the first real dispatch.
