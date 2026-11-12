# 3. Performance & Memory

Most n8n performance incidents aren't about volume. They come from **multiplication** (item cardinality) and **synchronisation** (everything firing on the same minute).

---

## 3.1 The 1.46-million-item cascade

**Symptom:** a monitoring workflow failed every 10 minutes after ~100 seconds. That looked like a timeout. The real error, `Node ran out of memory`, only showed up with `includeData=true`.

**Cause:** **a Data Table (or HTTP) node runs once per input item.** Three reads chained after a wide read:

```
Get Log Rows        3,014 rows
Get Recovery Log    3,014 × 2   =     6,028
Get Mute Rows       6,028 × 243 = 1,464,804   💥
```

**Blast radius:** an OOM in the n8n process every 10 minutes also produced `Database is not ready` 503s and **dropped roughly 1–2% of inbound webhooks** from *other* workflows. Two independent measurement methods agreed: several customer messages were being lost every day.

**Fix:** collapse reads that don't depend on the input:
- **`executeOnce: true`** on the node is a one-field fix when the read doesn't use the incoming item. On another workflow it cut run size by **~97%** (from ~180 MB per run to ~6 MB), and cut stored execution data from ~18 GB/day to under 1 GB/day.
- Otherwise, read once and join in a single Code node.

**Detect it early:** for every read node, ask *how many items arrive here?* Check input cardinality, not parameters.

> Bonus: the same bug inflated a monitoring alert's numbers ×100, and repeated pagination of a table under concurrent writes caused `result count changed during pagination`.

---

## 3.2 Daily heap OOM on a box with free RAM

**Symptom:** webhooks lost before any execution starts. In the container log:

```
Runner took too long to acknowledge acceptance of task
stream is not readable
Error in handling webhook request POST /webhook/...
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

**Cause:** Node's default `heap_size_limit` was **~2 GB on an 8 GB machine** with most of the RAM free. GC pauses grow as the heap fills, block the event loop, and incoming request bodies can't be read. `restart: always` hides the crash, because the container is back in ~20 seconds.

**Fix (environment):**

```
NODE_OPTIONS=--max-old-space-size=5120     # leave ~2.5 GB for OS + runner
N8N_DEFAULT_BINARY_DATA_MODE=filesystem    # PDFs/images out of the heap
N8N_RUNNERS_TASK_REQUEST_TIMEOUT=120000    # queue instead of dying at 60 s
```

---

## 3.3 Runner saturation is a scheduling problem

**Symptom:** Code nodes fail with `Task request timed out after 60 seconds`.

**What the logs showed:** `Offer expired - not accepted within validity window` appeared **almost an order of magnitude more often** than visible failures. The API undercounts starvation.

**Cause:** execution starts per minute, relative to the average:

| minute | load |
|---|---|
| :00 | **~4×** |
| :30 | ~3× |
| :45 | ~3× |

`minutesInterval` / `hoursInterval` schedules always fire from :00 and **can't be phase-shifted**.

**Fixes, cheapest first:**
1. Move high-frequency schedules to **cron expressions with odd offsets** (:02, :07, :13…).
2. Switch Code nodes from `runOnceForEachItem` to `runOnceForAllItems`. **Each item is a separate runner task**, but the execution view shows it as one run. A backup job went from hundreds of runner tasks to 2, and ran ~7× faster.
3. Remove a per-file loop that could be a single pass. Together, steps 2 and 3 cut **~40% of all daily Code tasks**.
4. Add `retryOnFail` to pure Code nodes (no static data). This is cheap insurance for customer-facing paths.
5. Don't simply raise runner concurrency on 2 vCPU. You get context switching, not throughput.

---

## 3.4 Measuring a huge SQLite DB without scanning it

When `execution_data` reaches tens of GB:

```sql
-- Full scan of the blob (minutes, may never finish):
SELECT SUM(octet_length(data)) FROM execution_data;

-- Index seek, reads only record headers (seconds):
SELECT SUM(octet_length(data)) FROM execution_data
WHERE executionId IN (SELECT id FROM execution_entity);

-- On-disk size per table, fast when constrained by name:
SELECT SUM(pgsize) FROM dbstat WHERE name = 'execution_data';
```

In our case **2 workflows accounted for ~95% of a ~60 GB table.**

Remember that pruning frees pages *inside* the file. With `auto_vacuum=0`, the disk only shrinks after `VACUUM`, which needs free space roughly equal to the *compacted* size.
