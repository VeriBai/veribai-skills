---
name: integrar-veribai
description: Integrate the VeriBai API (VeriFactu for the Spanish AEAT, and TicketBAI for the Araba, Bizkaia and Gipuzkoa foral treasuries) into the user's invoicing software, or review an existing integration. Use it when the user wants to issue, cancel or correct invoices with VeriBai (emitir, anular o rectificar facturas), have VeriBai render the invoice PDF with its QR, receive its webhooks, check a record's status, or when the code already calls sandbox.veribai.com, api.veribai.com or manage-api.veribai.com, or uses the veribai Python package.
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
| Emitting the invoice and getting the PDF (VeriBai renders it) | https://veribai.com/docs/api/documentos.md |
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
  For software serving many small businesses there are two valid models; ask which one:
  - **One account, the end users registered as issuers (clientes)**, the agency model. One
    key, one account; on LIVE every issuer needs a signed representation.
  - **One software, many VeriBai accounts.** Each end user opens their own VeriBai account,
    pastes their own API key into the software, and every invoice is self-issued by that
    account. The software holds **one key per customer, never shared across customers**.
    Verify each key at startup with `GET /v1/cuenta` and label it by what it returns
    (`nifClientePrincipal`, `entorno`), never by what the user typed. Quota and throttling are
    per key. Rotation and revocation happen in each customer's own dashboard, so the software
    needs a "paste a new key" screen; a key change must never require a deploy.
- **Which system.** Decided by where each issuer pays tax, not by the integrator: VeriFactu
  (AEAT) or TicketBAI (and which province). Sending to the other system's endpoint returns
  `400 TAX_SYSTEM_MISMATCH`. Software serving issuers of both needs both paths.
- **Who builds the invoice document.** Two paths; settle this before writing any code.
  - **(a) The user's software renders the invoice.** It computes the record, submits it with
    `POST /v1/verifactu/crear` or `POST /v1/ticketbai/crear`, gets the QR back (`consultar.md`)
    and prints it on the document it hands to the customer. The software is the invoicing
    software and owns the PDF.
  - **(b) VeriBai emits the invoice and returns the PDF.** The software sends the **commercial**
    lines (concept, quantity, unit price without VAT, discount %, fiscal keys per line) to
    `POST /v1/documentos/emitir`; VeriBai derives the fiscal record, files it, renders the PDF
    with the official QR already placed and returns it as `pdfBase64` (`api/documentos.md`).
    Choose this path when the user does not want their software to act as the invoicing
    software, has no PDF engine, or wants the regulated QR placement handled for them. Same API
    key, same host per environment; it works for the account's own NIF and for any registered
    issuer (B2B2C). The issuer block on the PDF (name and fiscal address) comes from the
    issuer's client row in VeriBai, never from the request body.
  Do not build a PDF generator for a user who wants path (b), and do not route a user who
  already renders invoices through path (b): the paths are alternatives, not layers.
- **Which operations.** Submissions, corrective invoices, cancellations, simplified invoices,
  foreign customers. Read the page for each.
- **Language and stack.** In Python, use the official `veribai` package (`pip install veribai`):
  it already handles retries, formats, waiting for the verdict, and webhooks. In any other
  language, call HTTP directly from the docs and apply the rules below yourself. The Python
  package does not wrap `/v1/documentos` yet: for path (b), call the route with plain
  `requests` and keep using the package for everything else.

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

**The QR must reach the end customer.** On path (a) the user's software prints it on the
invoice it hands over; how to get it: `consultar.md`. On path (b) it is already inside the PDF
VeriBai returns; the software only has to deliver that PDF.

**If VeriBai emits the invoice (path b, `/v1/documentos`).** The page has the fields; these are
the rules an integrator does not infer from it.
- **The body is the commercial document, never the record.** Lines with unit prices and
  percentages go in; `desglose`, `totales`, `cuota`… come out. A stray record field is a `400`
  naming it. Do not pre-compute VAT in the code: the API is the one rounding authority, and the
  `resumen` it returns is what to cross-check your own figures against.
- **`previsualizar` first, then `emitir`.** The preview returns a watermarked draft, persists
  nothing and consumes no number: use it to show the document before committing. Emission
  is the point of no return.
- **`estado = pdf_pendiente` is a `200` and the invoice IS filed**; only the PDF is missing.
  Call `POST /v1/documentos/{idFactura}/reanudar`. Never emit again.
- **`502 EMISION_INCIERTA`: outcome unknown, the body carries `idFactura`.** The only correct
  retry is `reanudar` with that id. Re-sending `emitir` with the byte-identical body is also
  safe (same invoice, `yaExistente: true`, no second number), but a body that changed in any
  way is a **new invoice**.
- **Numbering is automatic.** `numero` absent means the next number of the serie; the serie
  also counts invoices filed through `crear`, so do not mix your own counter with it. A refused
  emission consumes no number.
- **The PDF comes back as base64 in the JSON**, is stored, and can be downloaded again with
  `GET /v1/documentos/{idFactura}`. In Sandbox it is marked as having no fiscal validity.
- **`409 EMISOR_ADDRESS_MISSING` is a state problem, not a body problem.** The issuer's client
  row has no complete fiscal address, so the PDF cannot be rendered; nothing was persisted
  (the body carries `nifEmisor`). Retrying the same body is pointless: fix the row with
  `PATCH /v1/clientes/{nif}` (`direccion`, see `api/clientes.md`) and retry. Surface it to
  whoever manages the issuer, not to the person invoicing.
- **Series and recipients are managed in the dashboard, not by API.** The integrator keeps
  recipients in its own system and sends them inline in `destinatario`.
- Everything above about verdicts still applies: the emission is accepted, and the tax
  authority's answer arrives later by webhook or status query.

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
