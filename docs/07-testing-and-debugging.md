# 7. Testing & Debugging

---

## 7.1 The executions list hides running executions

`GET /executions` doesn't return executions that are still running or waiting, and the `status=waiting` filter can come back empty for executions that *are* waiting. A webhook test looks like it never fired.

**Fix:** read the **instance-wide** newest execution ID before triggering (no `workflowId` filter, because other workflows consume IDs quickly), then probe `GET /executions/{id}` across that range. Better still, poll whatever the run **writes** (a table row, a CRM field).

`status=new` was also observed returning the same rows as no filter at all. If filtered and unfiltered IDs match, the filter is being ignored.

---

## 7.2 A zero or a perfect percentage is a bug signature

During one investigation, three measurements were wrong:
- "0 messages waiting": the date parser threw on every row inside a bare `except`.
- A confident "most replies are bursts" that did not survive a recheck.
- "Hundreds lost, 100%".

**Rule:** before trusting a measurement, make it flag a **known-bad case**. If it can't find the bug you already know exists, it can't be trusted on the ones you don't.

---

## 7.3 A clone cannot reproduce timing

Isolated clones are good for proving logic. They can't prove anything that depends on a read happening before a write, because a 6-node clone has none of the latency of a 70-node production workflow. Race-sensitive behaviour (dedup, debounce, locks) has to be proven **in the live workflow**.

---

## 7.4 Live end-to-end tests with fake identities

The test harness used for every chatbot change:

1. **Seed:** create a fake CRM lead and state row with a reserved fake phone number.
2. **Drive:** POST provider-shaped webhook payloads to the *live* webhook, multi-turn, with realistic timing.
3. **Read:** pull the bot's reply from the execution data. Sending is stubbed or goes to a controlled number.
4. **Verify the side effects:** CRM status, notes, table rows. Not just the reply text.
5. **Clean up:** delete the seeded rows, verified with a per-key query, not a page scan.

A fix isn't done until it passes live, with the execution ID recorded.

> Sending to a fake number on some WhatsApp providers creates an "unconfirmed" contact that can't be deleted. Prefer stubbing the send, or a real number you control.

---

## 7.5 Assumption ledger before declaring a root cause

Before writing "root cause: X", list every assumption X depends on and mark each one:

| assumption | status | evidence |
|---|---|---|
| The workflow was deleted | ❌ UNVERIFIED | it was archived |
| The chat is muted forever | ❌ UNVERIFIED | mute auto-expires |
| The duration matches a timeout | ❌ wrong | the real error was OOM, visible only with `includeData=true` |

Ask about every UNVERIFIED line before you act. All three wrong diagnoses above came from skipping this step.

---

## 7.6 Diff against point-in-time backups

n8n's built-in version history may only capture some edit paths. Keep scheduled database backups. When behaviour changes without errors, diff the node parameters (model names, prompts, connections) against the last known-good snapshot **before** you start theorising.
