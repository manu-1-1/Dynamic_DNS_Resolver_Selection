# Adaptive, Privacy-Aware Resolver Scheduling for Oblivious DNS

*A plain-language deep dive into the problem, why current systems fall short, and how this idea mitigates it.*

---

## 1. Background: How DNS Privacy Evolved

### 1.1 Traditional DNS
When you type `www.amazon.com`, your device sends a **DNS query** asking "what's the IP address for this?" In classic DNS, this query goes in plaintext to your ISP's resolver.

**What the ISP's resolver knows:**
- Your IP address
- Every website you visit
- When you visited
- How often

Everything is visible. No privacy at all.

### 1.2 DNS over HTTPS (DoH)
DoH encrypts the query using HTTPS so it can't be read in transit (e.g., by someone on your Wi-Fi). It's a proposed IETF standard (RFC 8484, Oct 2018). A related protocol, **DNS over TLS (DoT)**, does the same thing with a different encryption/delivery method.

```
Browser → Encrypted HTTPS → Resolver (e.g., Cloudflare)
```

**The catch:** the resolver decrypts the query. So the resolver now sees:
```
Client IP → amazon.com
```
It knows **both** who you are **and** what you're visiting. That's the core privacy problem DoH doesn't solve.

### 1.3 Oblivious DNS over HTTPS (ODoH)
ODoH splits the pipeline by inserting a **proxy**:

```
Client → Proxy → Resolver
```

- **Proxy** sees: your IP + an encrypted blob it *cannot* decrypt.
- **Resolver** sees: the decrypted domain (`amazon.com`) but **not** who sent it.

Identity and query content are separated. This is a real, meaningful privacy improvement — and it's already standardized/deployed in some form (Cloudflare, Apple, Firefox).

---

## 2. What ODoH Still Doesn't Solve

ODoH fixes *identity-at-query-time*. It does **not** fix what happens when many queries accumulate at the same resolver over time. Five distinct problems remain:

### 2.1 Trust Concentration
If millions of users route through one resolver (e.g., Cloudflare), that resolver becomes a giant database of domain lookups — even if it can't name individual users, it becomes a massive point of trust and a high-value target.

### 2.2 Single Point of Failure
```
Client → Cloudflare → Internet
```
If Cloudflare goes down, DNS resolution for everyone routed through it stops.

### 2.3 Malicious/Compromised Resolver
A resolver could lie:
```
amazon.com → 54.x.x.x   (correct)
amazon.com → attacker.server   (malicious)
```
Encryption prevents *eavesdropping*, not *lying*. Users could be silently redirected to fake sites.

### 2.4 Traffic Analysis
Even fully encrypted, the **size and timing** of packets leaks information. Researchers have shown that websites can sometimes be inferred purely from encrypted traffic patterns (e.g., opening YouTube produces a distinctive small→large→medium packet sequence).

### 2.5 Profile Concentration & Re-Linking (the deepest issue)
This is the subtle one, worth walking through carefully.

**Step 1 — Silent profile building.**
Even without knowing your name, if all your queries go through the same resolver via the same encrypted session/connection, the resolver can link them together as belonging to *one anonymous entity*:

```
9:00am → connection #4471 → bank.com
9:03am → connection #4471 → gmail.com
2:00pm → connection #4471 → hospital.com
9:45pm → connection #4471 → youtube.com
```

No name attached — but a rich, continuous behavioral profile exists under that anonymous ID.

**Step 2 — The "one slip."**
At some point, *something* leaks identity alongside a timestamp/domain — a smart TV falling back to plaintext DNS, a misconfigured app, a public Wi-Fi fallback, or simply a lawful subpoena to an ISP asking "who had this IP at this timestamp?"

**Step 3 — Correlation.**
```
Plaintext leak: IP 203.0.113.42 → bank.com @ 9:00:01am
Resolver log:   Connection #4471 → bank.com @ 9:00:01am
```
Matching timestamp + domain links the two records.

**Step 4 — Retroactive exposure.**
Once `#4471 = IP 203.0.113.42` is established, **everything** that resolver ever logged for #4471 — bank, hospital, email, entertainment, months of history — becomes attributable to a real identity, all at once. No decryption was needed. Just one correlation event.

This doesn't require hacking. It can happen through:
- A technical fallback slip (one plaintext query)
- A subpoena matching two lawful log sets by timestamp

**The core insight:** the danger isn't just "can someone read my DNS traffic" — it's "how much does any single point in the system get to accumulate about me, such that one correlation event exposes everything at once?"

---

## 3. Why "Always Pick the Best Resolver" Fails

Suppose you have four resolvers, each with a trust score:

| Resolver | Trust Score |
|---|---|
| A | 95 |
| B | 90 |
| C | 82 |
| D | 75 |

A naive scheduler rule — "always pick the highest-trust resolver" — sends **100% of traffic to A, forever**. This isn't a bug in A; it's a structural consequence of a greedy rule. Over time, A silently reconstructs your entire behavioral profile anyway, recreating the exact centralization problem ODoH was meant to solve, just with better encryption wrapped around it.

A dumb fix — round-robin rotation (A→B→C→D→A...) — spreads load evenly but is *sensitivity-blind*: it treats a YouTube lookup and a bank lookup identically, and doesn't check whether a resolver has recently accumulated a lot of sensitive knowledge before handing it another sensitive query.

---

## 4. The Idea: Sensitivity-Aware, Knowledge-Decaying Resolver Scheduling

### 4.1 What's genuinely new here
Not the multi-resolver idea itself (ODoH-style resolver pools already exist). The novel parts are:

**(a) Query-sensitivity-aware routing.**
Not all domains carry equal risk:

| Domain | Sensitivity |
|---|---|
| youtube.com | Low |
| netflix.com | Low |
| bank.com | Very High |
| apollohospital.com | Very High |
| uidai.gov.in | High |

The scheduler treats these differently — high-sensitivity queries get routed more conservatively (weighted toward privacy/diversity over speed) than low-sensitivity ones.

**(b) Per-resolver accumulated-knowledge penalty.**
Instead of just tracking *recency* (round-robin) or *static trust* (greedy), the scheduler tracks **how much sensitive information a given resolver has already accumulated recently**, and penalizes it dynamically — even if it's the "best" resolver on paper.

```
Effective Score = Trust Score − Diversity Penalty
```

Example:

| Resolver | Trust | Penalty | Effective Score |
|---|---|---|---|
| A | 95 | 20 (handled last 20 queries) | 75 |
| B | 90 | 0 | 90 |
| C | 85 | 0 | 85 |

Next query goes to B. As different resolvers get used, penalties shift dynamically, spreading traffic naturally — and doing so *faster* for sensitive queries than for trivial ones.

### 4.2 The scoring model (conceptual)
```
Decision = f(Trust, Privacy, Latency, Load, Resolver Diversity, Query Sensitivity)
```
Rather than "which resolver is most trusted?", the question becomes: "which resolver gives the best overall outcome while keeping knowledge distributed and matching this query's sensitivity level?"

---

## 5. How the Idea Mitigates Each Problem

| Problem | Mitigated? | How |
|---|---|---|
| **Trust concentration** | ✅ Solved | No single resolver accumulates the full picture — traffic is actively spread, weighted by sensitivity |
| **Re-link blast radius** | ✅ Substantially reduced | If one slip correlates identity to a resolver, that resolver only ever held a *slice* of your history (e.g., just entertainment queries), not everything. An attacker would need to independently correlate *each* resolver separately to rebuild the full profile |
| **Single point of failure** | ✅ Solved | Load and availability are already inputs to the scoring function, so the scheduler naturally routes around a dead resolver |
| **Malicious/lying resolver** | ⚠️ Partially mitigated | Limits exposure (a rogue resolver only sees a fraction of traffic) and trust score should degrade quickly on bad responses — but doesn't by itself *detect* lying. Would need response-validation across resolvers as a separate mechanism |
| **Traffic analysis (packet timing/size)** | ❌ Not addressed | Splitting queries across resolvers doesn't change the packet-level pattern visible to an observer watching the device's outbound traffic before it even reaches a proxy. This needs a different mechanism (e.g., traffic padding/shaping) and should be stated explicitly as future work, not overclaimed |

### The key mental model: **blast radius reduction**
This idea does not prevent a correlation/leak event from ever happening (you can't fully control a device falling back to plaintext DNS, or a lawful subpoena). What it does is ensure that **when** such an event happens, it exposes only a fraction of a user's history rather than the complete picture — because no single resolver was ever allowed to accumulate everything in the first place.

---

## 6. Honest Scope Statement (important for credibility)

A strong research write-up says clearly what the idea does *and doesn't* solve — reviewers respond far better to this than to overclaiming:

- **Solves:** trust concentration, single point of failure, and meaningfully reduces re-link blast radius.
- **Partially solves:** damage from a malicious resolver (limits exposure, doesn't detect deception on its own).
- **Does not solve:** network-level traffic analysis/fingerprinting — a separate threat model requiring separate defenses.

---

## 7. One-Sentence Contribution Statement

> *We propose a resolver-selection scheduler for oblivious DNS that incorporates per-resolver accumulated knowledge as a dynamic, decaying privacy cost, and adjusts routing weights based on individual query sensitivity — reducing worst-case knowledge concentration per resolver compared to greedy-trust or round-robin baselines, while remaining upfront about the threats (malicious resolvers, traffic analysis) it does not fully address.*

---

## 8. Suggested Next Steps

1. **Literature check** — search "oblivious DNS resolver selection," "DNS privacy load balancing," "k-resolver DNS," "resolver diversity DNS privacy" to confirm exact positioning against prior work.
2. **Formalize the leakage metric** — pick a precise definition (e.g., Shannon entropy of the category-distribution of domains seen by each resolver in a sliding window) rather than a vague "knows too much."
3. **Build a simulator** — synthetic browsing trace (mixed sensitivity domains, realistic query rates), N simulated resolvers, compare your scheduler against round-robin and greedy-trust baselines.
4. **Metrics to report** — max knowledge concentration per resolver, average latency overhead, load variance across resolvers.
5. **Write the limitations section early** — the malicious-resolver and traffic-analysis gaps should be explicit, scoped-out items, not silent omissions.
