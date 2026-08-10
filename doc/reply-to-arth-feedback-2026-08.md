# Reply to Arth's Feedback — Infra Headroom (2026-08)

Hi Arth,

Thanks for framing this as a deliberate choice — happy to pick one.

**Recommendation: Option B — measure headroom directly during the pilot phase using the baseline script.**

Reasoning:

- The monitoring baseline script (Item 5) already captures exactly what this checkbox needs — CPU, memory, queue depth, job duration, error rate — labeled and comparable (`normal-baseline`, `peak-during-locust`, etc.). Running it during the actual pilot gives us those numbers from the **real production environment under real traffic**, which is a stronger, more direct answer to "does the system have headroom at 2,000 orders/day" than a resized test droplet could give us — no extrapolating from a synthetic box to guess at production behavior.
- Resizing the droplet (Option A) means cost, a founder approval cycle for the plan upgrade, and a wait — and even after all that, it still only gives us a *simulated* answer. The pilot-phase measurement is coming either way as part of Step 3, so Option B gets us the real number without adding a procurement step in between.
- This keeps us moving straight toward the pilot instead of pausing on an approval cycle for a data point the pilot will give us anyway.

If a resize happens to get approved for other reasons down the line, we're glad to grab a droplet-based reading too as a bonus data point — but we don't think it should block anything, since Option B answers the real question regardless.

**Why this doesn't need to repeat:** the headroom question resolves naturally as part of the pilot rollout itself (Step 3), which was always the next step regardless of this decision — we're only deciding *where* that measurement happens, not adding new work or a recurring check.

Best,
Viral
