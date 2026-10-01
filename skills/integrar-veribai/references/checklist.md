# VeriBai integration checklist

Go through it in order. Each point says what to check and why it matters. The concrete facts
are in the linked docs; if anything here contradicts them, the docs win.

## A. Can misfile an invoice, or lose it

1. **`200` is not treated as filed.** There is a second state (registered / rejected), received
   by webhook or by querying the status, and the system stores it.
2. **Rejections reach a person.** A `factura.rechazada` (or a rejected status) creates an alert
   or a visible task, not just a log line. Nothing automatically resends the same data after a
   rejection.
3. **The request is persisted before it is sent.** After a network failure or a restart, the
   same request is resent (same serie, number and date, same amounts), not a recomputed one.
4. **`409 INVOICE_IDENTITY_CONFLICT` is not retried**, and is handled as a numbering error.
5. **Retries follow the error table** (https://veribai.com/docs/api/errores.md): branch on
   `code`; `4xx` is not retried without a fix; `500`s marked non-retryable (e.g.
   `XML_PERSIST_ERROR` on a TicketBAI submission) are not retried.
6. **Numbering is sequential with no gaps, per serie**, and numbers are never reused. If another
   system came before, a new serie is used.
7. **No amount goes through `float`.** Decimal type end to end; sent as a string; no silent
   rounding.
8. **The issue date is computed in `Europe/Madrid`**, formatted `DD-MM-YYYY`.
9. **Each issuer uses the system that applies to it** (VeriFactu, or TicketBAI and province),
   and is registered as a client of the account.

## B. Webhooks

10. **The signature is verified over the raw body**, before parsing, with a constant-time
    comparison. An invalid signature is rejected without processing.
11. **Deliveries are deduplicated on `idEntrega`.** Receiving the same delivery twice does not
    repeat the work.
12. **The endpoint responds `2xx` within 10 seconds** and processes in the background.
13. **There is a plan for delivery gaps**: after an outage, verdicts are recovered with
    `GET /v1/registros?veredictoDesde=`, and someone is alerted if the webhook is suspended.

## C. Keys and environments

14. **No key in the code or the repository.** Environment variables or a secrets manager.
15. **The environment is chosen explicitly** in configuration, and startup checks the key with
    `GET /v1/cuenta` (and logs which environment it is in).
16. **There is a rotation procedure**: every service sharing the key is updated within the
    overlap window, and a compromised key is rotated **and** revoked
    (https://veribai.com/docs/autenticacion.md).

## D. What the end customer sees

17. **The QR is printed on the invoice** the customer receives
    (https://veribai.com/docs/consultar.md).
18. **Validation errors are shown with their field** (`errors[]`), so whoever invoices can fix
    them.
19. **No fiscal rules are copied into the code** that could block invoices the API would
    accept. Format validation only.

## E. Before LIVE (the person decides and does it)

20. **The integration has been through Sandbox**: submissions, the rejection cases,
    cancellation or corrective invoices if used, and webhook reception.
21. **Every issuer has a signed representation**
    (https://veribai.com/docs/api/representacion.md); without it, LIVE returns
    `403 REPRESENTATION_PENDING`.
22. **The production serie starts clean**: the LIVE chain begins with the first real invoice
    (https://veribai.com/docs/entornos.md).
