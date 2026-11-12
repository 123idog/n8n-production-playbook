# n8n Production Playbook

**Lessons from running 100+ live n8n workflows: multilingual WhatsApp AI agents, CRM sync, lead intake and call analysis, all in production with real customers.**

Most n8n content shows how to *build* a workflow. This playbook covers what happens **after** it goes live: silent failures, green executions that do nothing, memory blowups, race conditions, and API behaviour the docs don't mention.

Every entry here was found in production, measured, and fixed. Each one follows the same format:

> **Symptom** → what you actually see
> **Cause** → what is really going on
> **Fix** → what worked
> **Detect it early** → how to catch it before your customers do

---

## Chapters

| # | Chapter | What's inside |
|---|---|---|
| 1 | [Deploying via the API](docs/01-deploying-via-api.md) | PUT ignored on active workflows, UI saves overwriting API edits, settings whitelist, MCP tools dropping connections |
| 2 | [Data Tables](docs/02-data-tables.md) | 50-row default, filters vs. matchingColumns, broken timestamp filters, `isNotEmpty` matching `""` |
| 3 | [Performance & Memory](docs/03-performance-and-memory.md) | 1.46M-item cascade, heap OOM on an 8 GB box, runner saturation on round minutes, `executeOnce` |
| 4 | [AI Agents & LLM nodes](docs/04-ai-agents-and-llm.md) | `$fromAI` arity, silent model fallback, few-shot feedback loops, durable chatbot memory |
| 5 | [Webhooks & Messaging](docs/05-webhooks-and-messaging.md) | Redelivery and dedup, restart floods, "sent" vs. delivered, sandboxed webhook HTML |
| 6 | [Salesforce Integration](docs/06-salesforce-integration.md) | Outbound Message ACK, queue lag, phone matching, converted leads |
| 7 | [Testing & Debugging](docs/07-testing-and-debugging.md) | Hidden running executions, "a zero is a bug signature", clones can't reproduce timing |

---

## The five rules that cover most of this

1. **A green execution proves nothing.** Check what the workflow *wrote*, not whether it errored.
2. **Check input cardinality, not parameters.** Most n8n nodes run once per input item. A read placed after a node that emits 3,000 items runs 3,000 times.
3. **Verify every deploy with a GET.** A 200 from the API doesn't mean the change landed.
4. **Last writer wins, whole document.** n8n has no merge and no conflict detection. Diff live before you overwrite.
5. **Test live, with fake identities.** Isolated clones prove logic, never timing.

---

## Stack these lessons come from

n8n (self-hosted, v2.x, SQLite, task runners) · WhatsApp Business API via a BSP · Salesforce · Gemini / OpenAI · Telegram · Data Tables · Claude Code as the build-and-deploy agent.

## About

Written by **Ido**, AI & Automation Engineer. I design and run production AI agents and automation systems for sales and customer-service teams.

[LinkedIn](https://www.linkedin.com/in/ido-guy)

*All examples are generalised. No client data, credentials, prompts or identifiers are included.*
