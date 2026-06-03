# Debugging Reference — Systematic Approaches by Bug Category

Use this during Phase 1 (bug map) and Phase 3 (investigation plan) when the user reports a bug. Match the symptom to a category, then apply the corresponding investigation strategy.

---

## How to Use This Reference

1. Read the user's bug description
2. Match to the closest category below
3. Use the investigation strategy to build the Phase 1 map
4. Use the confirmation steps in Phase 3 to guide the developer toward confirming root cause before writing a fix

**Never assume root cause from symptoms alone.** The categories below are starting hypotheses, not diagnoses.

---

## Category: Wrong Output / Incorrect Behavior

**Symptom:** The code runs without error but produces wrong results.

**Probable causes (in order of likelihood):**
1. Wrong assumption about input data shape or values
2. Off-by-one in indexing or iteration
3. Mutation of shared state (a function modified something it shouldn't have)
4. Incorrect conditional logic (wrong operator, wrong comparison, wrong precedence)
5. Silent type coercion (int/float, string/bytes, etc.)

**Investigation strategy:**
1. Add logging or print statements at the input and output boundary of the suspected function — confirm what data is actually flowing in and out
2. Trace backward from the wrong output to find where it first diverged from the expected value
3. Isolate the function producing the wrong output and test it in isolation with known input
4. Check every conditional that controls the path leading to the wrong output

**Confirmation before fixing:** reproduce the wrong output with a minimal, controlled input. If you can't reproduce it reliably, you don't understand it yet.

---

## Category: Crash / Exception

**Symptom:** The process raises an exception and stops (or is caught somewhere upstream).

**Investigation strategy:**
1. Read the full stack trace — the bottom frame is where it crashed, the top frame is where the exception was caught or propagated from
2. Find the frame closest to your own code (not in a library) — that's usually where the real problem is
3. Check what value caused the exception — `None` where something was expected, index out of range, type mismatch
4. Ask: why did that value arrive here? Trace it backward to its origin.

**Common crash causes:**
- `AttributeError: 'NoneType' has no attribute X` → a function returned `None` instead of the expected object; the real bug is upstream
- `KeyError` → a dict key is assumed to exist but doesn't; find where the dict is built and why the key is missing
- `IndexError` → an index is assumed valid but isn't; check the length assumption
- `RuntimeError` / framework-specific errors → read the message carefully; framework errors are usually informative about what precondition was violated

---

## Category: Works Locally, Fails in Production / Other Environment

**Symptom:** Code behaves correctly in one environment and incorrectly in another.

**Probable causes:**
1. Environment-specific configuration (different env vars, config files, secrets)
2. Different dependency versions
3. Different hardware or OS behavior (file paths, endianness, GPU vs. CPU)
4. Timing differences (production load exposes race conditions or timeouts)
5. Data differences (production data has edge cases local data doesn't)

**Investigation strategy:**
1. List every difference between the two environments: OS, hardware, Python/Node/etc version, dependency versions, environment variables, available GPU/memory
2. Check whether the bug is data-dependent — does it happen on all inputs or specific ones?
3. Check whether the bug is load-dependent — does it only appear under concurrent requests or high throughput?
4. Compare config files and env vars between environments

---

## Category: Race Condition / Intermittent Bug

**Symptom:** The bug appears sometimes but not reliably; hard to reproduce.

**Probable causes:**
1. Shared mutable state accessed from multiple threads or processes without proper locking
2. Timing-dependent behavior (a result used before an async operation completes)
3. Uninitialized state read before it's written
4. External system state (DB, cache, queue) that can be in an intermediate state

**Investigation strategy:**
1. Add logging with timestamps around the suspected race — look for interleaving patterns
2. Check every piece of shared state in the affected code — what reads it, what writes it, from which threads/processes, with what locking
3. Look for `await` / `async` calls where a result is used before the awaited operation completes
4. Check for `time.sleep()` or polling loops that assume a condition will be true "soon enough"

**Note:** Race conditions are the hardest category to fix by intuition. Prefer adding explicit synchronization (locks, queues, events) over "just sleeping longer" — the latter hides the race without fixing it.

---

## Category: Memory Leak / Resource Leak

**Symptom:** Memory or other resource usage grows over time; eventually crashes or slows dramatically.

**Probable causes:**
1. Objects held in a long-lived collection that are never removed
2. Event listeners or callbacks registered but never deregistered
3. File handles, database connections, or sockets opened but not closed
4. Circular references preventing garbage collection (rare in Python with the GC, but possible with `__del__`)

**Investigation strategy:**
1. Identify what resource is leaking — memory, file handles, DB connections, GPU memory
2. Find every place in the code where that resource is allocated/acquired
3. Verify that every allocation has a corresponding deallocation — check exception paths too (use context managers)
4. For long-lived collections (caches, registries), check whether old entries are ever evicted

---

## Category: Performance Degradation

**Symptom:** The system becomes slow over time or under load.

**Probable causes:**
1. N+1 query pattern (a query inside a loop where a single query would work)
2. Unnecessary recomputation (a result recalculated on every request instead of cached)
3. Blocking I/O in an async context (a synchronous operation blocking the event loop)
4. Missing index on a frequently queried database column
5. Growing queue / buffer that is processed slower than it fills

**Investigation strategy:**
1. Profile before optimizing — measure which operation is slow, don't assume
2. Look for loops that make external calls (DB, API, filesystem) on every iteration
3. Check whether the slowdown correlates with data volume, request count, or uptime

---

## Category: Silent Failure (No Output, No Error)

**Symptom:** The system runs without error but nothing happens — no output, no change, no event.

**This is the hardest category.** Something is silently swallowed.

**Investigation strategy:**
1. Add logging at the entry point of the expected behavior — confirm execution reaches it
2. Walk forward from the entry point, adding logs at each branch until you find where execution diverges from expectations
3. Check every `except` block — look for bare `except:` or `except Exception: pass` that swallows errors silently
4. Check every early `return` and `continue` — something may be exiting early
5. Check conditional guards that may be preventing execution (wrong flag, wrong state, wrong data)

**The fix is never "add more retries" until you understand why it's failing silently.**
