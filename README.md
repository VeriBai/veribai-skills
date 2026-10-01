# VeriBai skills

*Agent Skills for integrating the VeriBai API (Spanish VeriFactu and TicketBAI e-invoicing) with
Claude Code, Codex, Cursor, Copilot, Gemini CLI and other agents. Docs in Spanish.*

[Agent Skills](https://agentskills.io) para integrar la API de [VeriBai](https://veribai.com)
(VeriFactu y TicketBAI) en tu software con ayuda de un agente de IA: Claude Code, Codex,
Cursor, GitHub Copilot, Gemini CLI y cualquier agente compatible con `SKILL.md`.

| Skill | Qué hace |
| --- | --- |
| [`integrar-veribai`](skills/integrar-veribai/SKILL.md) | Guía la integración: qué sistema aplica, cómo tratar el veredicto de la administración, reintentos seguros, importes y fechas, webhooks y el paso a producción. Se activa sola cuando trabajas con VeriBai. |
| [`revisar-veribai`](skills/revisar-veribai/SKILL.md) | Revisa una integración existente contra la [checklist](skills/integrar-veribai/references/checklist.md) y te dice qué cambiar, con fichero y línea. Se invoca a mano. |

## Qué hacen y qué no

- **Enseñan, no ejecutan.** Las skills no llaman a la API de VeriBai, ni siquiera a Sandbox. El
  agente escribe el código; lo ejecutas tú. Un alta en Sandbox deja un registro de prueba
  permanente en la cadena del emisor, así que eso lo decides tú.
- **No sustituyen la documentación, la enlazan.** Los campos, códigos y límites se leen de
  [veribai.com/docs](https://veribai.com/docs) ([`llms.txt`](https://veribai.com/llms.txt)),
  que es la fuente de verdad. Las skills aportan el criterio: lo que un integrador suele hacer
  mal y cómo evitarlo.
- **Nunca trabajan contra producción.** El paso a LIVE es tuyo.

## Instalación

### Claude Code

```
/plugin marketplace add VeriBai/veribai-skills
/plugin install veribai@veribai
```

Quedan disponibles como `/veribai:integrar-veribai` y `/veribai:revisar-veribai`; la primera
también se carga sola cuando el agente detecta que estás integrando VeriBai. Para actualizar:
`claude plugin update veribai@veribai`.

### Otros agentes

Copia las dos carpetas de [`skills/`](skills/) al directorio de skills de tu herramienta.
Instálalas **juntas**: `revisar-veribai` usa la checklist de `integrar-veribai`.

| Herramienta | Documentación |
| --- | --- |
| Codex / ChatGPT | https://developers.openai.com/codex/skills/ |
| Cursor | https://cursor.com/docs/context/skills |
| GitHub Copilot / VS Code | https://docs.github.com/en/copilot/concepts/agents/about-agent-skills |
| Gemini CLI | https://geminicli.com/docs/cli/skills/ |
| Claude (claude.ai, API) | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview |
| Otros | https://agentskills.io |

## Relacionado

- [Documentación de la API](https://veribai.com/docs)
- [Cliente oficial de Python](https://github.com/VeriBai/veribai-python-client) (`pip install veribai`)
- [Soporte](https://veribai.com/docs/soporte)

## Licencia

[Apache 2.0](LICENSE).
