# 1. Deploying via the API

If you build n8n workflows programmatically (REST, MCP, or an AI coding agent), the deploy path is where most silent damage happens.

---

## 1.1 PUT to an active workflow returns 200 and changes nothing

**Symptom:** `PUT /workflows/{id}` returns 200. A later GET shows the **old** nodes. Every "fix" you deploy seems to revert.

**Cause:** while a workflow is active, the public API accepts the request but doesn't apply structural node changes.

**Fix:** always deploy in this order:

```
GET (backup) → POST /deactivate → PUT → POST /activate → GET (verify)
```

**Detect it early:** never trust the PUT response. Compare the fields you changed in the post-deploy GET with what you sent.

> ⚠️ **Prove the write path before deactivating.** If the deactivate succeeds and the PUT or re-activate is refused (permissions, auth, schema), production stays down while you debug. Do a no-op PUT first. If a write fails mid-deploy, **re-activate first, investigate second.**

---

## 1.2 A UI save silently reverts your API edits

**Symptom:** a prompt change deployed via API and verified by GET disappears hours later. Other recent changes survive.

**Cause:** saving in the editor writes the **entire workflow** from the copy the browser loaded. There's no field-level merge and no optimistic-concurrency check. Anything deployed after that tab opened is overwritten. Changes from before the tab opened survive, which makes the loss very hard to spot.

**Fix:**
- Close or refresh editor tabs for any workflow you're about to edit via API.
- Track `updatedAt` and node count. An `updatedAt` you didn't cause means someone saved over you.
- Keep every deployed prompt and Code node in a file, so re-applying takes one command.

---

## 1.3 Diff the live version before you overwrite it

**Symptom:** you deploy your local prompt file and wipe out a fix that someone else (or another agent session) deployed that morning.

**Fix:** before each deploy, GET the live workflow and compare the fields you're replacing with **your last deployed version**. If they differ, apply your change as a minimal patch onto the live text. Never replace the whole field.

**Encoding trap:** sending large RTL (Hebrew/Arabic) JSON through some shells (Git Bash `curl`, Windows PowerShell 5.1) corrupts multibyte characters into `U+FFFD`. Upload with a script that builds the bytes explicitly (`Buffer.from(JSON.stringify(body), 'utf8')`), then verify byte equality and a count of zero for `�` in the edited fields.

---

## 1.4 The public API rejects unknown `settings` keys

**Symptom:** `request/body/settings must NOT have additional properties`, even for a one-character change.

**Cause:** the editor stores internal settings keys (e.g. `binaryMode`) that the public API schema rejects. Tools that round-trip the whole workflow through the public API fail on every save.

**Fix:** PUT only `{ name, nodes, connections, settings }`, and whitelist `settings` to:

```
executionOrder, timezone, saveDataErrorExecution, saveDataSuccessExecution,
saveManualExecutions, saveExecutionProgress, executionTimeout, errorWorkflow
```

---

## 1.5 Programmatic edits can silently drop connections

**Symptom:** after a parameter-only edit to a Code node, a downstream node never runs. The connection is now `"main": [[]]`.

**Observed with:** MCP partial-update operations (`updateNode`, and even one `patchNodeField` on `jsCode`), and occasionally on workflow *creation*.

**Fix:**
- After every programmatic edit, fetch the structure and assert that every expected connection still exists.
- Re-assert critical connections explicitly at the end of an edit batch.
- When one tool fails in an odd way, switch to a plain REST PUT. Don't retry the same tool.

**Related blocker:** some tools validate the **whole** workflow before saving and refuse when there are pre-existing orphan nodes unrelated to your change. Scan for orphans before patching.

---

## 1.6 Don't hard-delete workflows that ran recently

**Symptom:** after deleting a temporary test workflow, the Insights feature fails with `Could not find shared workflow` and stays broken until a restart.

**Cause:** Insights events are buffered in memory per workflow ID. Flushing a buffer for a deleted workflow fails and the buffer is re-queued forever.

**Fix:** archive or deactivate and rename (`ZZZ-temp-…`). Don't hard-delete.

---

## 1.7 Deploys wipe in-memory chat memory

**Symptom:** a chatbot "forgets" conversations after a deploy.

**Cause:** buffer-window memory lives in RAM per workflow version. Every deactivate→PUT→activate cycle resets it. Loading an empty session can also return **zero items**, which silently stops downstream chains.

**Fix:** keep conversation state in a durable store (see [4.4](04-ai-agents-and-llm.md#44-durable-chatbot-memory-without-buffer-memory)), deploy conversational bots in quiet hours, and set `alwaysOutputData` on memory loads.
