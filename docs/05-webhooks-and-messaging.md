# 5. Webhooks & Messaging

Messaging providers retry, batch and redeliver. Your workflow has to be idempotent, or customers notice.

---

## 5.1 Providers redeliver: dedup on the message ID

**Symptom:** the bot answers the same customer message twice, sometimes an hour apart. To the customer it looks like the bot is talking to itself.

**Cause:** the WhatsApp provider redelivers any webhook it doesn't consider delivered. The workflow had no "already handled" check.

**Fix:** a seen-ID gate at the top of the flow:

```
Normalize (extract msgId) → Lookup msgId → Dedup Gate → Mark Seen → ...rest
```

- **Look up the single incoming ID** (`msg_id eq {{ $json.msgId }}`). Don't bulk-read the seen table.
- **Mark before doing any work**, so a duplicate that arrives during the first run loses.
- **Fail open:** if the seen list can't be read, or a message has no ID, let it through. A rare duplicate is better than silently dropping customers.

> 🔴 **The isolated clone passed 4/4 and production failed.** The first version bulk-read the whole seen table. In a 6-node clone that was instant. In the 70-node production workflow, two runs 8 seconds apart both read before either wrote, about a 25-second window. **Clones prove logic, never timing.** See [7.3](07-testing-and-debugging.md#73-a-clone-cannot-reproduce-timing).

---

## 5.2 Restarting n8n causes a message flood

**Symptom:** after a container restart, customers get bursts of replies to messages they sent long ago. One bot handled dozens of turns within a few minutes.

**Cause:** webhooks are refused for 30–60 seconds. The provider doesn't drop them. It retries, and the whole backlog lands at once.

**Rules:**
- Never restart during business hours.
- Expect the same burst when you **re-activate** a bot that was switched off.
- After a restart, watch executions per minute for 10 minutes.
- Message-ID dedup (5.1) and a "too old to answer" cutoff keep floods harmless.

---

## 5.3 "Sent" isn't "delivered"

**Symptom:** a campaign logs a customer as messaged and permanently excludes them, but they never got anything.

**Cause:** the send-template endpoint returned `result: true` for a number that isn't on WhatsApp. The real signal is `validWhatsAppNumber`.

```
fake number -> { result: true, validWhatsAppNumber: false, contactStatus: "UNCONFIRMED" }
real number -> { result: true, validWhatsAppNumber: true,  contactStatus: "VALID" }
```

**Fix:** use three states, never a boolean:

```js
const accepted  = resp.result === true || resp.ok === true;
const badNumber = resp.validWhatsAppNumber === false;
const status = !accepted ? 'failed' : (badNumber ? 'invalid_number' : 'sent');
```

---

## 5.4 A webhook delay can look like a silent bot

**Symptom:** the bot goes quiet for 5 minutes mid-conversation.

**Diagnosis in two queries:**
1. Compare the message `created` timestamp in the webhook body with the execution's `startedAt`. Here two messages created 90 seconds apart started **in the same second**, nearly 5 minutes late. The provider released a batch.
2. Check whether other conversations ran during the gap. If they did, the stall is provider-side and per-conversation.

**Design lesson:** a debounce that steps aside because it "already saw a newer message in history" has an unbounded failure mode: if that webhook never arrives, nobody answers. Add a sweeper for conversations whose last message is inbound and unanswered after N minutes.

---

## 5.5 A safety net that ran 382 times and never fired

**Symptom:** a "rescue unanswered customers" workflow ran every 10 minutes for a week (hundreds of green executions) and logged **0 recoveries**.

**Cause:** **an HTTP Request node replaces its input item with the response.** The next Code node read `item.json.phone`, which no longer existed, so every candidate was discarded on line one.

**Fix:** take identity fields from the paired upstream item, and **abort if item counts ever differ**. Never risk sending one customer's message to another's number. Add a fire cap and a dry-run flag.

**Second bug, same workflow:** the code assumed a 5-minute cadence, but the cron fired every 10. The slice index advanced by 2 per run, so with an even slice count **half the pool was never checked.**

---

## 5.6 HTML served from a webhook is sandboxed

Pages returned by *Respond to Webhook* are served with a `sandbox` CSP that doesn't include `allow-same-origin`. Accessing `localStorage`, `sessionStorage` or `document.cookie` throws a `SecurityError` and **stops the rest of the script**. Buttons render but do nothing.

**What still works:** inline scripts, the DOM, `fetch()` to the same n8n host, and the URL fragment via `history.replaceState`. Render session state into the HTML on the server.

---

## 5.7 The Telegram node defaults to HTML parse mode

Setting `parse_mode` to `""` doesn't disable it. Any `<`, `>` or `&` in model output fails the send with `can't parse entities`. HTML-escape free text before sending.

A Telegram Trigger created via the API also needs an explicit `webhookId`, or the webhook registers at a path that returns 404.
