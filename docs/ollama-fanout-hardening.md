# Hardening an n8n → local-LLM step: bounded fan-out and capped generation

A field note from running a self-hosted n8n workflow that calls a **local Ollama
model** behind a small HTTP wrapper. The symptom looked like a timeout; the cause
was a queue. This is the write-up of the diagnosis and the two changes that fixed
it — both are generic and apply to any "n8n fan-out → single-threaded model
server" path.

## Symptom

The LLM step (an `httpRequest` node POSTing to a local service that wraps Ollama)
failed after **exactly `300078 ms`** with:

```
The connection was aborted, perhaps the server is offline
```

Every upstream node succeeded. Only the LLM step failed, and always at almost
exactly 300 s — i.e. right at a configured node timeout. The obvious move is to
raise the timeout. That is the wrong move, and here is why.

## Investigation

The same HTTP call, run by hand from the host, returned in **24 s**. So the
service was healthy. Two measurements from the service's own logs cracked it:

1. **Far more requests than items.** Dozens of requests arrived within the same
   second, though the workflow was only processing a handful of items.
2. **Climbing durations.** Those requests did not fail fast. Their service-side
   durations increased across the batch:

   ```
   47s  74s  102s  ...  288s
   ```

   That ramp is the signature of **queueing**, not of one slow call.

Two faults were stacking:

### Fault 1 — a generation cap that was far too generous

The wrapper passed `num_predict: 8192`. On a long prompt the 30B model rambled
for **~272 s** on a single call. Capping generation at `3072` brought the same
call to **~24 s** — roughly a 10× win, with no meaningful quality loss for the
task. A short prompt had masked this: a quick test finished in ~36 s, which made
the long path look fine.

### Fault 2 — unbounded parallel fan-out onto a single-threaded backend

This was the real killer. The workflow's trigger carried a **parameter** (a
"dry-run" flag) that was meant to shrink the batch. But a trigger node's
*parameters are metadata, not item data* — they never appear in `$json`. So the
downstream expression read `undefined`, the batch size silently fell back to its
default of **50**, and the workflow fanned all ~46 results into the LLM step **at
once**. Ollama serializes generation, so 46 concurrent requests became a queue
whose tail exceeded the 300 s limit. The node that "timed out" was simply last in
line.

## Fix

Cap generation and bound the batch.

Make the generation cap a knob with a sane default (wrapper side):

```js
num_predict: Number(process.env.NUM_PREDICT || 3072)
```

Bound the fan-out (workflow side). Note the comment — this is the part that bites:

```js
// Trigger parameters are NOT delivered to the workflow as item data,
// so an expression like $json.dry_run silently resolves to undefined and
// falls through to the default. Hard-code the batch for the manual path.
limit: 2
```

## Verification

- The generation step dropped to ~70 s for a real item; the whole 7-node run
  finished **green in ~80 s**, every node `success`.
- The service log confirmed the downstream render steps returned `200` in under a
  second each — the queue was gone.

## Rules of thumb

- A failure that lands on a suspiciously round number (300 s) is a **hint, not a
  cause**. A wall-clock value that matches a configured limit exactly means "the
  line snapped here", not "this is the limit's fault."
- **Look for the queue.** Many near-simultaneous requests with *climbing*
  durations mean a slow backend serializing a fan-out. Fix the fan-out and the
  generation cap — not the timeout.
- **Trigger parameters are not item data.** Reading a config value off the
  trigger inside `$json` resolves to `undefined`. Set batch size explicitly, or
  pass it through a real input item.
- **Bound concurrency to the backend.** A single-threaded model server needs a
  batch size *and* a concurrency limit, not just a longer timeout.
