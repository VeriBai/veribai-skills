---
name: integrar-veribai
description: Integrate the VeriBai API (VeriFactu for the Spanish AEAT, and TicketBAI for the Araba, Bizkaia and Gipuzkoa foral treasuries) into the user's invoicing software, or review an existing integration. Use it when the user wants to issue, cancel or correct invoices with VeriBai (emitir, anular o rectificar facturas), receive its webhooks, check a record's status, or when the code already calls sandbox.veribai.com, api.veribai.com or manage-api.veribai.com, or uses the veribai Python package.
license: Apache-2.0
---

# Integrate VeriBai

VeriBai is a REST API that validates, signs, chains and submits invoicing records to
**VeriFactu** (AEAT) and **TicketBAI** (the foral treasuries of Araba, Bizkaia and Gipuzkoa).
This skill tells you **how** to integrate it correctly. The **facts** (fields, codes, limits)
live in the public documentation: read them there, never from memory.

Talk to the user in their language. The docs, the API's field names and its error messages are
in Spanish; keep identifiers exactly as written.

## Where things are

Full documentation, as markdown for agents: **https://veribai.com/llms.txt**. Every page has a
`.md` version. Read the relevant page before writing code for that endpoint; the contract
overrides anything you remember or assume.

| You need | Page |
| --- | --- |
| Overview and first submission | https://veribai.com/docs/introduccion.md |
| Keys, environments, rotation | https://veribai.com/docs/autenticacion.md, https://veribai.com/docs/entornos.md |
| Which system applies to each issuer | https://veribai.com/docs/verifactu-o-ticketbai.md |
| Self-issuing or issuing for third parties | https://veribai.com/docs/modelo-b2b2c.md |
| VeriFactu submission, field by field | https://veribai.com/docs/crear-factura-verifactu.md, https://veribai.com/docs/api/verifactu.md |
| TicketBAI submission, field by field | https://veribai.com/docs/crear-factura-ticketbai.md, https://veribai.com/docs/api/ticketbai.md |
| Fixing (subsanar), cancelling, corrective invoices | https://veribai.com/docs/rectificar-anular.md |
| Dates | https://veribai.com/docs/fechas.md |
| Status, QR, XML, listings | https://veribai.com/docs/consultar.md, https://veribai.com/docs/api/facturas.md |
| Errors and what to retry | https://veribai.com/docs/api/errores.md |
| Limits, pagination, conventions | https://veribai.com/docs/api/limites.md |
| Webhooks | https://veribai.com/docs/api/webhooks.md |
| Registering issuers (clientes) | https://veribai.com/docs/api/clientes.md |
| Representation before the tax authority | https://veribai.com/docs/api/representacion.md |
| Official Python client | https://veribai.com/docs/cliente-python.md |

If a page contradicts this skill, **the page wins**; tell the user.

## Your limits as an agent

An invoicing record cannot be deleted. Even in Sandbox, every submission is sealed into the
issuer's hash chain and reaches the tax authority's test environment.

- **Never submit invoices yourself.** You write the code; the person runs it. If they
  explicitly ask you to run a test against Sandbox, first remind them that it leaves a permanent
  test record and spends their key's quota.
- **Never against LIVE.** Do not use LIVE keys, do not switch a configuration to `live`, and do
  not run anything against `api.veribai.com`. Going to production is the person's decision, made
  in their configuration.
- **Never put a key in code** or in a versioned file. Use environment variables or a secrets
  manager.
- **In examples and tests, use placeholder NIFs**: issuer `B00000000`, recipient `B11111119`.
  Unit tests mock HTTP; they do not call the API.

## How to work

### 1. Understand the case before writing code

Find out, asking if needed:

- **Who issues.** The user's own company (self-issuing), or the companies it serves (issuing
  for third parties)? Every issuer must be registered under their account, or the API returns
  `403 UNAUTHORIZED_NIF`. Each NIF has its own chain, independent of the others.
- **Which system.** Decided by where each issuer pays tax, not by the integrator: VeriFactu
  (AEAT) or TicketBAI (and which province). Sending to the other system's endpoint returns
  `400 TAX_SYSTEM_MISMATCH`. Software serving issuers of both needs both paths.
- **Which operations.** Submissions, corrective invoices, cancellations, simplified invoices,
  foreign customers. Read the page for each.
- **Language and stack.** In Python, use the official `veribai` package (`pip install veribai`):
  it already handles retries, formats, waiting for the verdict, and webhooks. In any other
  language, call HTTP directly from the docs and apply the rules below yourself.

### 2. Apply the rules almost every integrator skips

These are about **semantics**, not validation; no code generator infers them from the schema.

**`200` means _accepted_, not _filed_.** The tax authority's response, the one with legal
effect, comes later. Design for two moments: accepted, then registered or rejected.
- To learn the outcome, use **webhooks** (`factura.registrada`, `factura.rechazada`,
  `factura.anulada`). Polling `GET /v1/facturas/{idFactura}/estado` is fine to start, but every
  call spends quota. In Python: `client.facturas.esperar_verdicto(...)`.
- To recover verdicts for a period with no deliveries, use `GET /v1/registros?veredictoDesde=`.
- **A rejected invoice is NOT filed**, and the obligation is the issuer's. A person must be
  told. **Never automatically resend the same data**: the result is the same rejection, and in
  TicketBAI every attempt advances the chain for good. Fix it with a correction (subsanación) or
  a corrective invoice (see `rectificar-anular.md`).

**An invoice's identity is issuer + serie + number + issue date (`fechaExpedicion`).**
- An identical resend does not duplicate: it returns `200` with the existing record. That is
  why a submission can be retried after a network failure. Do not add your own idempotency key.
- The same identity with different data (`importeTotal`, `tipoFactura`…) is
  `409 INVOICE_IDENTITY_CONFLICT`: another invoice with an already-used number. **Do not
  retry**; it is a numbering bug to fix.
- **Persist the exact request (and its number) before sending it.** If the process dies without
  knowing whether it arrived, resend that same request, not a recomputed one.
- Production numbering is **sequential with no gaps, per serie**. When migrating from another
  system, start a new serie.

**Retry only what the docs mark as retryable.** Always branch on the error's `code` field,
never on `message` or on the HTTP status alone. `429` with exponential backoff; `4xx` is not
retried without fixing the request; some `500`s are **not** retryable (e.g.
`XML_PERSIST_ERROR` on a TicketBAI submission). The full table is in `api/errores.md`:
implement it from there.

**Amounts: `Decimal`, never `float`, sent as strings.** In floating point 0.1 is not 0.1, and
one cent off in a tax amount is a fiscal error. Never round silently: if an amount does not fit
the format, fail and let the person decide.

**Dates: `DD-MM-YYYY`, and "today" in peninsular Spanish time (`Europe/Madrid`).** A future
`fechaExpedicion` is rejected; between 00:00 and 02:00 Madrid time the UTC date is still
yesterday. Compute the date in `Europe/Madrid`, not in UTC or server time.

**Do not copy fiscal rules into the code.** Serialize types correctly and let the API be the
authority: it returns `400` with `errors[]` naming the field. Show that detail to the end user.
Local fiscal validation goes stale as soon as an authority changes a rule, and blocks invoices
the API would accept.

**Webhooks: verify the signature over the raw bytes, before parsing.**
- `X-VeriBai-Signature` is `sha256=` + HMAC-SHA256 of the **raw body** with the webhook secret.
  If the framework already parsed the JSON, those bytes are useless: read the raw body. Compare
  in constant time.
- Delivery is at-least-once: **deduplicate on `idEntrega`** (equal to `X-VeriBai-Delivery-Id`),
  which is identical on every retry.
- Respond `2xx` within 10 seconds and process in the background. After 20 consecutive failures
  the webhook is suspended.
- In Python: `veribai.webhooks.parse_entrega(cuerpo, cabeceras, secreto=...)` verifies and
  parses in one call.

**The QR must reach the end customer.** It is printed on the invoice the issuer hands over. How
to get it: `consultar.md`.

**Environments: develop in Sandbox.** Base URL `https://sandbox.veribai.com`; the management API
(issuers, webhooks, representation) is at `https://manage-api.veribai.com` with the same key.
Keys are per environment; the API's environment identifier is `test`. In Python, pass
`environment=` explicitly. Check the key at startup with `GET /v1/cuenta`.

### 3. Prepare the move to production, without making it

Finish with the checklist in [references/checklist.md](references/checklist.md): go through it
with the user point by point and tell them what is still open. Also mention what is not code:
in LIVE every issuer needs a **signed representation** (`403 REPRESENTATION_PENDING` if
missing; see `api/representacion.md`).

## Reviewing an existing integration

If the code already exists, do not rewrite it: **review it against
[references/checklist.md](references/checklist.md)**. For each point, say whether it holds,
where (file and line), and what you would change. Start with what can misfile or lose an
invoice: retries, identity, `float` amounts, webhook signature, rejections nobody handles.
Cosmetics last.
