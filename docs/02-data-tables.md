# 2. Data Tables

n8n Data Tables are an excellent built-in store for dedup logs, bot memory and campaign state. Their failure modes are almost all **silent**.

---

## 2.1 `get` returns only 50 rows by default

**Symptom:** a "process all pending rows" job processes exactly 50, every time.

**Fix:** set `returnAll: true` on any `get` that drives a batch. The same applies to every API you query. Always set explicit limits and pagination, and before analysing anything state *retrieved N of M*.

---

## 2.2 `update` / `delete` / `get` need `filters`, not `matchingColumns`

**Symptom:** runtime error `At least one condition is required`, even though validation passed.

**Fix:**

```json
"matchType": "allConditions",
"filters": { "conditions": [
  { "keyName": "id", "condition": "eq", "keyValue": "={{ $json.id }}" }
]}
```

`columns.mappingMode: "autoMapInputData"` is still the right choice for the SET values.

---

## 2.3 Timestamp-as-string filters only work at day granularity

Measured on an ISO timestamp column (`2026-08-26T05:12:23.798Z`):

| filter | result |
|---|---|
| `gt '2026-08-26'` | ✅ correct |
| `gt '2026-08-26T05:00:00'` | 🔴 34 rows, 28 of them **older** than the cutoff |
| `gt '2026-08-26T05:00:00.000Z'` | 🔴 always 0 rows |

**Fix:** filter coarsely by date in the node, then apply the real time window in a Code node. Better still, store times as **epoch milliseconds in a number column**, where `gt` works correctly.

---

## 2.4 `isNotEmpty` matches empty strings

A row with `transcript: ""` matches `transcript isNotEmpty`. Use `condition: "neq", keyValue: ""` instead. This matters for retention jobs: without it, every run re-touches every old row.

---

## 2.5 Update and upsert are partial

Columns you don't send keep their stored values. They aren't nulled. Verified on a 37-column row where 7 fields were sent and the other 30 stayed byte-identical.

**Use it:** leave a column out of the mapping to protect a hand-maintained field from an automated re-ingest.

---

## 2.6 Dedup pattern for outbound campaigns (safe restart, no double-send)

```
Query → Prepare (normalise + in-batch dedup via Set)
      → rowNotExists filter (ONE call, BEFORE the loop)
      → Loop → Send → Tag result → Insert log row (successes only) → Loop
```

- `rowNotExists` works as a filter on its own. No IF node is needed.
- **Keep it outside the loop.** Inside a SplitInBatches loop, an iteration where every item is filtered out returns empty and **the whole loop stalls silently**.
- Log **only confirmed successes**, so failures retry on the next run.

---

## 2.7 Public REST API limitations

- No REST endpoint deletes or updates rows (405/404). Use the MCP data-table tools or the node.
- `sortBy` on row listing is **ignored**: you get the oldest rows. Verifying "no test rows left" by scanning the latest page gives a false clean result. Always verify deletion with a per-key filter.
- Pagination is cursor-based. `offset` and `skip` are ignored.

---

## 2.8 Paired-item resolution hangs after a Data Table node

**Symptom:** a Code node after a Data Table read runs for minutes and then shows "Task cancelled", while its output displays the *input* items.

**Cause:** `$('Webhook').item` triggers paired-item resolution back through the chain, and the Data Table node breaks the pairing.

**Fix:** read local data (`$json`, `$input.all()`), or move the logic to a point before the Data Table read. If you must reach back, `$('X').first()` avoids paired-item resolution.
