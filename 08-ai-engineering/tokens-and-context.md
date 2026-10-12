# Tokens and context windows
Last revised: 2026-10-12 · Confidence: 3/5

## 60-second explanation (say this in an interview)

An LLM reads and bills in **tokens**, not words. A token is a piece of text, roughly ¾ of an English word. Code, JSON and numbers use more tokens.

The **context window** is the most tokens one request can hold, **input and output together**: system prompt, chat history, tool definitions, tool results and the answer. The API is **stateless**, so every call resends the whole history. Cost grows each turn, and long chats must be trimmed or summarised.

If input plus output don't fit, the answer gets **cut off, not shortened**. Quality also drops before the window is full ("context rot"), so a bigger window is not a reason to fill it.

## How it works underneath

**Tokens**
- A tokenizer splits text into tokens before the model sees it. It is like the `javac` lexer turning `int balance = 100;` into `int`, `balance`, `=`, `100`, `;`. The split is based on how often pieces appear in training text, not on grammar.
- Input tokens and output tokens are billed separately. Output usually costs more.
- The model never sees single letters, so it is weak at counting letters or reversing strings.

**Context window = a fixed-size buffer**
- It is like a fixed-capacity `ByteBuffer`. The system prompt, history, tool schemas and tool results are `put()` in first, and the answer goes into whatever space is left.
- `max_tokens` caps the **output only**. It does not make the window bigger.

**Stateless API = REST without `HttpSession`**
- The model remembers nothing between calls. Turn 20 sends turns 1–19 (your messages **and** the model's earlier answers) plus the new message, the system prompt and the tools.
- So turn 20 costs more than turn 2, even if both questions are one line.

**What happens when it doesn't fit**
- Input alone is bigger than the window → the API returns a 400 error ("prompt is too long") on every model.
- Input + `max_tokens` is bigger than the window:
  - Claude 4.5 and newer: the request is accepted, but generation stops when the window is full, with `stop_reason: "model_context_window_exceeded"`.
  - Older models: a validation error before anything runs.
- The model does **not** know its budget and does not shorten its answer to fit. It writes normally and gets cut off.

**Ways to keep the context small**
1. Summarise old turns (compaction).
2. Trim or drop old turns: keep only the last N, like a bounded queue.
3. Clear old tool results. Large JSON from 15 turns ago is dead weight (Anthropic has "tool result clearing" for this).
4. Make tools return summaries, not raw rows.
5. Send only the relevant documents with RAG (week 4).

## Code example (from my own project)

Example token budget for one Wallet assistant call. The numbers are made up, only to show the sizes:

| Part of the call | Tokens |
| --- | --- |
| Tool result: 200 transactions as JSON | 12,000 |
| Previous 6 turns of chat | 2,400 |
| 3 tool definitions | 900 |
| System prompt | 300 |
| Model's answer | 250 |
| **Total** | **≈ 15,850** |

Lessons:
- **Tool results are the big cost.** Return `{"food": 4820.50, "travel": 1200.00}`, not 200 rows.
- **Compute in code, explain in the LLM.** Java sums 200 numbers correctly every time. A model doing the arithmetic can get it wrong.
- **Tool definitions are paid on every call,** even when unused.

Logging token usage and the stop reason in Spring AI (TODO: replace with the commit link from my Wallet app):

```java
ChatResponse response = chatClient.prompt()
        .user(question)
        .call()
        .chatResponse();

Usage usage = response.getMetadata().getUsage();
log.info("tokens in={} out={} total={}",
        usage.getPromptTokens(), usage.getCompletionTokens(), usage.getTotalTokens());

String finishReason = response.getResult().getMetadata().getFinishReason();
if (!"STOP".equalsIgnoreCase(finishReason) && !"end_turn".equalsIgnoreCase(finishReason)) {
    log.warn("Answer may be cut off: finishReason={}", finishReason);
}
```

## Gotchas / where people get it wrong

- Thinking the model "remembers" the chat. It doesn't. Your code resends everything.
- Thinking `max_tokens` makes the window bigger. It only caps the output.
- Thinking the model shortens its answer when space is low. It doesn't. It gets cut off. **Always check the stop reason.** A cut-off answer can look like a complete one, like an unchecked partial read on an `InputStream`.
- Forgetting the model's own earlier answers are part of the resent history.
- Filling a big window "because it fits". Context rot lowers quality first.
- Dumping raw data into tool results. It costs more and the answers get worse.

## Interview questions I should be able to answer

1. Why does turn 20 of a chat cost more than turn 2, even if both questions are one line?
2. You have a 200k window, `max_tokens` = 4,000 and 199,000 input tokens. What happens, and what are three fixes?
3. Why should a tool return a summary instead of every row?
4. What is the difference between the context window and `max_tokens`?
5. How would you track and control token cost in a production LLM feature?

My checkpoint answers on 2026-10-12: Q1 4/5, Q2 2/5 (I thought the model would shorten its answer; it gets cut off instead), Q3 4/5.

## Links

- [Context windows – Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Context editing – Claude Platform Docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- Lesson doc: [Lesson: Tokens and context windows](https://claude.ai/code/artifact/f465af33-d955-4ece-a7f1-d016e94371a1)

<!-- revision-log.md row:
| 08-ai-engineering/tokens-and-context | 2026-10-12 | 2026-10-13 | 2026-10-19 | 2026-11-11 | 2027-01-10 | 3 |
-->
