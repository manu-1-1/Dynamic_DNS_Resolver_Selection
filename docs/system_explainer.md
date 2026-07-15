# Understanding the System: Query Sensitivity, Resolver Memory, and Adaptive Routing

*A step-by-step explainer of how the scheduling system works, piece by piece.*

---

## The Big Picture

We're building a system that decides, **for every single DNS query, which of several resolvers should handle it** — and the decision isn't random or fixed. It's based on:
1. How sensitive that specific query is
2. How much each resolver already knows (recently)
3. Normal operational factors (speed, reliability, load)

**Analogy:** You have 4 mailboxes (resolvers) you can send letters through. You don't want any one mailbox operator learning your entire life story, so you spread your letters across all 4 — but not evenly. Your bank statement goes somewhere careful; your junk mail can go anywhere.

That's the whole idea. Below is each piece, in detail.

---

## Piece 1: Query Sensitivity Label

### What it is
Every DNS query is a request for a domain (e.g., `bank.com`, `youtube.com`). Before deciding which resolver handles it, we first ask: **how sensitive/risky is this domain?**

### Why we need it
Not every website reveals the same amount about you if leaked.
- Someone learning you visited `youtube.com` → reveals almost nothing.
- Someone learning you visited `apollohospital.com` → could reveal a medical condition.

So queries can't all be treated the same. Each one needs a **label** describing how carefully it should be routed.

### The 4 buckets

| Level | Examples | Why |
|---|---|---|
| **LOW** | youtube.com, netflix.com, news sites | Casual browsing, low risk if exposed |
| **MEDIUM** | gmail.com, social media | Personal, but not extremely sensitive |
| **HIGH** | government portals, tax sites | Sensitive institutional data |
| **VERY_HIGH** | banks, hospitals, insurance | High-risk personal/medical/financial data |

### How labeling is actually implemented
The simplest approach — and the one used here — is a **rule-based lookup**, not machine learning:
- Check the domain name against known categories/keywords
- E.g., if the domain contains "bank" or matches a known banking domain list → label `VERY_HIGH`

This is intentionally simple and transparent. It's good enough for a first working version. A smarter, learned classifier is possible later, but that's future work — not part of the core contribution.

### Output of this step
Every query `q` gets a label:
```
S(q) ∈ {LOW, MEDIUM, HIGH, VERY_HIGH}
```
This label feeds into everything downstream.

---

## Piece 2: What Each Resolver "Knows" (the memory problem)

### What it is
Every resolver sees a stream of queries routed to it over time. We track: **how much sensitive information has this specific resolver accumulated recently?**

### Why it matters
If Resolver A handles your bank query, hospital query, and tax query all within the same hour, it now "knows too much" about you — even though each individual routing decision seemed reasonable in isolation. The system needs to notice this build-up and react to it *before* it becomes a full profile.

### How it's tracked
Each resolver keeps a **short memory window** — e.g., the last 50 queries it handled, or everything within the last hour. Old entries fall out of the window automatically over time. This is how "forgetting" happens — no complicated decay function needed, the sliding window does it naturally.

### Turning memory into a number
We look at what sensitivity labels exist inside a resolver's memory window and compute one score: **how much accumulated sensitive knowledge does this resolver currently hold.**
- Mostly LOW-sensitivity queries in memory → low score
- Several VERY_HIGH queries in memory → high score

This number becomes the **penalty** — the more a resolver already knows, the more it gets penalized for receiving *more* sensitive queries.

---

## Piece 3: Resolver Attributes (the "normal" operational stuff)

Separate from sensitivity and knowledge tracking, each resolver also has ordinary properties any load balancer cares about:

- **Trust score** — how reliable/reputable is this resolver
- **Latency** — how fast does it respond
- **Load** — how busy is it right now

These matter because the system isn't purely privacy-obsessed — it still needs to work well in practice. A system that maximizes privacy but is slow and unreliable isn't usable.

---

## Piece 4: The Decision — Combining Everything

For every incoming query, the decision process is:

1. **Classify the query** → determine its sensitivity label (Piece 1)
2. **Check every resolver's current state** → trust, latency, load, and accumulated sensitive knowledge (Pieces 2 & 3)
3. **Combine these into one score per resolver** — but the *way* they're combined shifts depending on the query's sensitivity:
   - **LOW sensitivity** (e.g., YouTube) → speed/load matter most; the knowledge penalty barely factors in
   - **VERY_HIGH sensitivity** (e.g., bank) → avoiding resolvers that already know a lot matters most; speed is de-emphasized
4. **Pick the resolver with the best combined score** for this specific query
5. **Update that resolver's memory** — it now "knows" this query too, which shapes future decisions

This is why the system is called **adaptive**: the same pool of resolvers is treated differently depending on what's being asked, and each resolver's internal state constantly shifts as new queries arrive.

---

## Piece 5: Why This Solves the Original Problem

**The original issue:** if you always pick "the best" resolver, that one resolver eventually learns your entire browsing history. If it's ever compromised, subpoenaed, or correlated with your identity through any side channel, everything leaks at once.

**With this system:** your bank queries might mostly land on Resolver C, hospital queries on Resolver D, YouTube on Resolver A — because the system actively avoids letting any single resolver accumulate too much sensitive knowledge.

**Result:** even in a worst-case leak or correlation event, whoever gets exposed only had a **slice** of your activity — not the complete picture. This is the core privacy benefit: reducing the *blast radius* of any single point of compromise.

---

## Piece 6: What Actually Needs to Be Built

| Component | Status | Notes |
|---|---|---|
| Sensitivity labeling rulebook | Simple, done conceptually | Domain/category → LOW/MEDIUM/HIGH/VERY_HIGH lookup table |
| Knowledge-tracking mechanism | Needs formalizing | Sliding window + a formula turning "recent activity" into one penalty number |
| Scoring/decision formula | Needs formalizing | Combines trust, latency, load, penalty — weights shift by sensitivity class |
| Simulation | To be built | Synthetic browsing trace + fake resolvers, compare this scheduler against "always pick best" and "round robin" baselines |

The knowledge-tracking formula and scoring function are the two places where rigorous math work adds the most value — this is where a strong formalization (entropy measures, weighted sums, or even a bandit/optimization framing) turns the concept into something provably sound rather than just intuitively reasonable.

---

## Quick Recap in One Paragraph

Every query gets labeled by how sensitive its domain is. Every resolver keeps a short memory of what sensitive stuff it's recently handled. When a new query comes in, the system scores each resolver using its trust, speed, load, and how much sensitive knowledge it's already accumulated — weighting these factors differently depending on how sensitive *this* query is. The resolver with the best score wins, and its memory updates. Over time, this naturally spreads sensitive queries across resolvers instead of letting one resolver build a complete profile, so if any single resolver is ever compromised, it only exposes a fraction of a person's activity.
