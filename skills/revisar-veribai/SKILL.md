---
name: revisar-veribai
description: Review an existing VeriBai API integration (VeriFactu and TicketBAI) against the best-practice checklist and say what to change, with file and line. Use it when the user asks to review, audit or validate their VeriBai integration code (revisar, auditar o validar la integración).
license: Apache-2.0
disable-model-invocation: true
---

# Review a VeriBai integration

This skill is the direct entry point to the review. The method and the checklist live in the
**`integrar-veribai`** skill, section "Reviewing an existing integration", and in its
`references/checklist.md` file. Load it and follow it.

If `integrar-veribai` is not installed, an equivalent public checklist can be derived from
https://veribai.com/docs/api/errores.md, https://veribai.com/docs/api/webhooks.md and
https://veribai.com/docs/entornos.md; but install both skills together.

Review rules:

- **Do not change code unless asked.** Deliver the report first.
- **Do not run anything against the API**, not even Sandbox, to check a point. Review by reading
  the code and configuration.
- **Order by severity**: first what can misfile or lose an invoice, then webhooks, keys and
  environments, cosmetics last.
- For each point: **holds / does not hold / not applicable**, where (file and line), and the
  concrete change you propose.
- Write the report in the user's language.

If the user named a specific part of the code, focus on it: $ARGUMENTS
