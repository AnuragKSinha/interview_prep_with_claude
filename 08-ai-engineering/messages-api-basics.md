# Messages API basics

Written 2026-10-20 (from a quiz). Revise on: 2026-10-21, 2026-10-27, 2026-11-19, 2027-01-18

## What it is
One HTTP POST to `/v1/messages`: you send the conversation, you get back one assistant reply.

## Request shape
- `model`: which Claude model to use
- `max_tokens`: required. A hard cap on the length of THIS reply
- `system`: top-level field, not inside `messages`
- `messages[]`: a list of `{role, content}`, where role is `user` or `assistant`, alternating

## Stateless
- The API keeps no memory between calls. The app stores the history and resends ALL of it on every call.
- Cause and effect: being stateless is WHY history piles up. The pile-up doesn't make it stateless.
- Cost: every call pays for the whole history again as input tokens, so call #20 costs more than #1. It's also slower, and a long chat can eventually hit the context window limit (fix: trim or summarize old turns).

## max_tokens ✏️ (gap)
- It applies to ONE response, NOT to the whole conversation. Each call gets its own cap.
- It's a ceiling, not a target: a higher value doesn't make Claude write more.
- If the reply hits the cap, it is cut off mid-sentence.

## stop_reason, and how I'd handle each
- `end_turn`: finished normally
- `max_tokens`: truncated. Raise the cap, or continue: send the partial reply back as the assistant turn and let Claude carry on.
- `stop_sequence`: hit a custom stop string I supplied
- `tool_use`: Claude wants me to run a tool (week 2)
- Never assume `end_turn`. Switch on it like an enum.

## Java/Spring analogy
Like a stateless REST service with no HttpSession: the client carries the state (`List<Message>`). `max_tokens` is like a read timeout: it limits the call, it doesn't guarantee completeness.

## Scenario: Spring Boot chat assistant for card customers ✏️ (gap)

### 1. Conversation id
- Thousands of chats run at once. Each chat gets an id (a UUID), and the history is stored under that key.
- First call: no id, so the server creates one and returns it. After that, the client sends the id with every message.
- Same idea as a session id or a shopping-cart id: the server stays stateless and looks the state up by key.

| conversationId | history |
|---|---|
| c-7f3a (Priya) | "What's my balance?" → "₹42,000" → "And last month?" |
| c-91bd (Rahul) | "Block my card" → "Which card ending?" → "4321" |

### 2. Request flow
```java
@PostMapping("/chat")
ChatResponse chat(@RequestBody ChatRequest req) {
    String id = req.conversationId() == null ? UUID.randomUUID().toString() : req.conversationId();
    List<Message> history = store.load(id);                     // 1. load THIS chat only
    history.add(new Message("user", mask(req.text())));         // 2. add the new turn (PII masked)
    Response r = claude.send(history);                          // 3. send the FULL history
    handle(r.stopReason());                                     // 4. never assume end_turn
    history.add(new Message("assistant", r.text()));
    store.save(id, history);                                    // 5. save it back
    log.info("conv={} model={} in={} out={} stop={} ms={}",     // 6. metrics, no raw content
             id, r.model(), r.usage().inputTokens(), r.usage().outputTokens(), r.stopReason(), r.latencyMs());
    return new ChatResponse(id, r.text());                      // client keeps the id
}
```

### 3. Store: local (to learn) → ConcurrentHashMap, not HashMap
- The controller/store is a singleton bean. Tomcat runs many request threads on the SAME map at the same time.
- HashMap isn't thread-safe: lost updates, entries missed during a resize (Java 7 could even loop forever), and NO error, just wrong data.
- `synchronizedMap` is safe but uses one lock for everything, so it's slow. ConcurrentHashMap gives lock-free reads and locks only per bucket on writes.
- Catch: the map is safe, but the `List` inside isn't (a double-tapped send). Update atomically per key:
```java
store.compute(id, (k, h) -> { var l = h == null ? new ArrayList<Message>() : new ArrayList<>(h); l.add(msg); return l; });
```
- Why Postman testing never shows the bug: requests are sequential, with no overlap. Race bugs depend on timing. Catch them with a concurrency test (ExecutorService + CountDownLatch, then check the size) or a load test (JMeter, Gatling, k6).

### 4. Store: production → Redis (or a DB)
- 3 instances behind a load balancer, each with its own memory. Rahul's msg 1 lands on A and msg 2 on B: B has no history, so Claude loses context. A restart also wipes in-memory chats.
- Fix: a shared Redis that all instances read and write (key `chat:<id>`). The instances become stateless and scale horizontally.
- Sticky sessions (always route Rahul to A) work, but the chat dies if A dies and the load spreads unevenly. Mention it, then avoid it.
- TTL = auto-expiry. Reset it on every message so the chat expires after 30 min idle:
```java
redisTemplate.opsForValue().set("chat:" + id, history, Duration.ofMinutes(30));
```
Why: bounded memory, PII isn't kept longer than needed, and a stale chat doesn't confuse the next one.
- Use Redis for short-lived, fast chats. Use a DB when chats must be kept (audit, compliance).
- Long chats: trim or summarize old turns, because cost, latency and the context window all grow with the history.

### 5. PII masking (payments company = key point)
- Mask card numbers, account numbers and similar data BEFORE storing or sending, e.g. `4111 1111 1111 4321` becomes `**** **** **** 4321`.
- Never log raw message content. Log only ids and metrics.

### 6. stop_reason handling
- `end_turn`: return the reply
- `max_tokens`: truncated. Raise the cap or continue (send the partial back as the assistant turn), and alert if it happens often
- `tool_use`: run the tool, send the result back (week 2)
- Anything unexpected: log it and return a safe fallback message

### 7. Log per response
conversation id, model, input_tokens, output_tokens, stop_reason, latency, request id. These give you cost, speed and truncation alerts.

### Interview answer (scenario, 30 seconds)
"Each chat has a conversation id. The service loads that chat's history from Redis, appends the new masked turn, sends the full history to the model, checks stop_reason, and saves the reply back with a 30-minute idle TTL. Instances stay stateless so they scale horizontally. Locally I'd use a ConcurrentHashMap with compute, because the bean is a singleton shared by all request threads. I log token usage, stop_reason and latency, never raw content."

## Interview answer (30 seconds)
"The Messages API is stateless, so my service stores the conversation per session and resends the full history on each call. That means cost and latency grow with the length of the chat, so I trim or summarize old turns. I check stop_reason on every response, because max_tokens means the reply was truncated, and I log token usage for cost tracking."

## Gaps from the quiz
1. Forgot `max_tokens` is a required request field.
2. Thought max_tokens was cumulative across the chat. It's per response.
3. Scenario: add the conversation id, PII masking, and logging stop_reason + latency.
4. Follow-ups covered: why ConcurrentHashMap (singleton bean, many threads), why Redis + TTL across instances.