# Deployment Strategies: Blue-Green vs Canary vs Rolling

All three solve the same problem: **move users from the old version (v1) to the new version (v2) with zero downtime.**
They differ in **how much risk you take at once** and **how fast you can undo it**.

```
                 Before            During                 After
Blue-green   [ v1 ██████ ]    [ v1 ██████ ]          [ v2 ░░░░░░ ]
                              (v2 env ready, idle)    100% flipped at once

Canary       [ v1 ██████ ]    [ v1 █████ | v2 ░ ]    [ v2 ░░░░░░ ]
                              ~5% users on v2         ramp 5 → 25 → 50 → 100%

Rolling      [ v1 ][ v1 ]     [ v2 ][ v2 ]           [ v2 ][ v2 ]
             [ v1 ][ v1 ]     [ v1 ][ v1 ]           [ v2 ][ v2 ]
                              replace in batches      all servers on v2

█ = old version (v1)    ░ = new version (v2)
```

---

## 1. Blue-Green

**How it works**
- Run **two full production environments**: Blue (live, v1) and Green (idle).
- Deploy v2 to Green and test it quietly (smoke tests, health checks).
- Flip the load balancer / DNS / router so **100% of traffic goes to Green in one step**.
- Keep Blue around; if something breaks, **flip back**.

**Pros**
- Near-instant rollback (just switch traffic back).
- Clean cutover; only one version serves users at a time (after the flip).
- Easy to test the new environment before it takes real traffic.

**Cons**
- **~2× infrastructure cost** while both environments exist.
- Every user hits v2 at the same moment, so a bug affects everyone.
- Stateful parts (DB, caches, sessions) are shared or need syncing, which is the hard part.

**When to use**: You need fast, reliable rollback and can afford duplicate infra (e.g. critical payment services).

---

## 2. Canary

**How it works**
- Route a **small slice of real traffic** (e.g. 1–5%) to v2; the rest stays on v1.
- Watch metrics: error rate, latency (p95/p99), CPU/memory, business KPIs (e.g. payment success rate).
- If healthy, ramp up gradually: 5% → 25% → 50% → 100%.
- If unhealthy, route the slice back to v1 (often automated).

**Pros**
- **Smallest blast radius**: a bug only hits a small % of users.
- Catches issues that only appear with real traffic and real data.
- Can be fully automated (progressive delivery: Argo Rollouts, Flagger, Spinnaker).

**Cons**
- Needs **strong observability** and automated analysis, or it's just slow rolling.
- Needs **weighted traffic routing** (service mesh like Istio/Linkerd, weighted LB, ingress rules).
- v1 and v2 run side by side for a while, so you need backward compatibility.
- Slower full rollout.

**When to use**: High-traffic, user-facing systems where you want to validate with real users before full release.

---

## 3. Rolling

**How it works**
- Replace instances **in batches**: take 1–2 out of the pool, upgrade to v2, health-check, put back, repeat.
- **Default strategy in Kubernetes Deployments.**
  - `maxSurge`: how many extra pods can be created above desired count during update.
  - `maxUnavailable`: how many pods can be down during update.

**Pros**
- **No extra infrastructure** needed.
- Simple; built into most orchestrators (Kubernetes, ECS, ASGs).
- Gradual, so not every server changes at once.

**Cons**
- **Slow rollback**: you have to roll everything back again.
- v1 and v2 run together during the rollout, so you need backward compatibility.
- **No control over which users hit which version** (unlike canary, which is traffic-based; rolling is instance-based).
- Capacity is reduced during the rollout if `maxSurge = 0`.

**When to use**: Default for most stateless services where cost matters and the risk is moderate.

---

## Quick Comparison

| | Blue-Green | Canary | Rolling |
|---|---|---|---|
| Rollback speed | Instant | Fast | Slow |
| Extra infra cost | High (2×) | Low–medium | None |
| Blast radius of a bug | All users | Small slice | Grows batch by batch |
| Traffic control | All-or-nothing switch | Fine-grained % | None (instance-based) |
| Monitoring needed | Helpful | **Essential** | Helpful |
| Versions live together | Briefly (or none) | Yes | Yes |
| Complexity | Medium | High | Low |

---

## Interview Talking Points (Senior Level)

### 1. Database compatibility: the #1 follow-up
In canary and rolling (and briefly in blue-green), **v1 and v2 run against the same DB at the same time**.
So schema changes must be **backward compatible**. Use the **expand-and-contract** (parallel change) pattern:

1. **Expand**: add the new column/table (nullable, no breaking changes). Old code still works.
2. **Migrate**: deploy code that writes to both old and new; backfill existing data.
3. **Switch**: deploy code that reads from the new column.
4. **Contract**: once no code uses the old column, drop it in a later release.

Never rename or drop a column in the same release that stops using it.

### 2. Canary vs feature flags
- **Canary** = gradual rollout of a **deployment** (new binary/code version).
- **Feature flag** = gradual rollout of a **feature** inside already-deployed code.
- They're complementary: deploy dark with a flag off, then canary the flag by user segment.

### 3. Canary vs A/B testing
- **Canary** goal: **is v2 safe?** (errors, latency).
- **A/B test** goal: **is v2 better?** (conversion, engagement); needs statistical significance.

### 4. What metrics decide a canary's fate
- Error rate (5xx), latency p95/p99, saturation (CPU, memory, connection pools).
- Business metrics: e.g. payment authorization success rate, checkout completion.
- Compare canary vs baseline **side by side**, not against a static threshold alone.

### 5. Stateful concerns
- **Sessions**: use sticky sessions carefully or externalize session state (Redis).
- **Caches**: v2 may write cache entries v1 can't read; version your cache keys.
- **Message queues / Kafka**: v1 consumers may receive messages produced by v2; keep message schemas backward/forward compatible (e.g. Avro/Protobuf with schema registry).
- **Long-running requests**: use connection draining / graceful shutdown before killing old instances.

### 6. A good one-liner answer
> "For a payments service, I'd use **canary** with automated metric analysis and instant rollback, combined with **expand-and-contract** DB migrations so v1 and v2 can safely coexist. For lower-risk internal services, **rolling** is usually enough. **Blue-green** fits when I need an instant, clean switch and can afford the duplicate infrastructure."

---

## Related Strategies (good to name-drop)
- **Recreate**: stop all v1, start v2. Has downtime; fine for dev/test.
- **Shadow / dark launch**: mirror real traffic to v2 but discard its responses; tests under real load with zero user impact.
- **A/B testing**: traffic split for measuring business impact, not safety.
