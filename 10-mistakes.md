# Mistakes file

Every wrong or incomplete answer, with the correct answer in my own words. Re-read before every mock. Newest first.

## 2026-10-12 · Week 1

### DSA: 1. Two Sum

**1. Integer overflow: didn't know how to check it.**
- `nums[i]` and `target` are in [-10⁹, 10⁹], so the max `target - nums[i]` = 10⁹ − (−10⁹) = 2,000,000,000.
- `Integer.MAX_VALUE` = 2,147,483,647, so it fits, with only about 147 million to spare.
- Java int overflow **wraps silently**. There's no exception, just a wrong value. If it could exceed the limit, use `long`.
- Habit: whenever I add, subtract or multiply two inputs, check the limits and **say it out loud** in the interview.

**2. Gave the output for the map-overwrite case, but not the why.**
- In `[2,2,5]` with target 7, the second `put(5, 1)` overwrites `put(5, 0)`.
- No answer is lost: a key is only overwritten by an index holding the **same value**, so either index makes a valid pair.
- Habit: for every edge case, give **output + reason**, not just the output.

**3. Complexity said without the reason.**
- Time O(n): one pass, each HashMap get/put is O(1) on **average**.
- Space O(n): the map holds up to n−1 entries.
- Follow-up: since Java 8, a crowded bucket becomes a red-black tree, so the worst case is O(log n) per lookup, not O(n).

**4. Code tidy-up: double lookup.**
- `containsKey()` then `get()` = two hash lookups. Use one `get()` and check for `null`.

### AI: Tokens and context windows

**5. Thought the model shortens its answer when the context is nearly full. It doesn't.**
- The model doesn't know its token budget. It writes normally and gets **cut off mid-sentence**.
- Input alone too big → 400 error ("prompt is too long").
- Input + `max_tokens` too big → newer Claude models accept the request but stop with `stop_reason: "model_context_window_exceeded"`; older models return a validation error.
- Habit: **always check the stop reason** in code. A cut-off answer can look like a complete one.

**6. Only knew one fix for an over-full context (summarising).**
- Summarise old turns (compaction).
- Trim or drop old turns: keep the last N, like a bounded queue.
- Clear old tool results (Anthropic's "tool result clearing").
- Make tools return summaries, not raw rows.
- Send only relevant documents with RAG.

**7. Small gap: what gets resent each turn.**
- The resent history includes the **model's own earlier answers**, not just my messages.
- The answer being generated now is output, billed separately. It isn't something I send.

**8. Small gap: why summary tools are better.**
- Not just tokens and cost. **Compute in code, explain in the LLM**: Java sums 200 numbers correctly every time, and a model can get the arithmetic wrong.
- Context rot: quality drops before the window is full.
