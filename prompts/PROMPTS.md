# ScyllaDB GTM Hunter — AI Prompts & Design Rationale

Three prompts power this system. Each was designed with specific principles — not just "what to say" but *why* the structure produces better outputs.

---

## Prompt A: Lead Scoring

**Node:** `AI — Score Lead (Claude)`  
**Model:** `claude-sonnet-4-20250514` | **Max tokens:** 500

```
You are a B2B sales intelligence analyst for ScyllaDB, a high-performance
NoSQL database that is a drop-in replacement for Apache Cassandra but offers
10x better throughput, zero GC pauses (written in C++), and 3-5x fewer nodes.

Score this lead's potential as a ScyllaDB prospect on a 0-100 scale.

LEAD PROFILE:
- Name: {{ $json.first_name }} {{ $json.last_name }}
- Role: {{ $json.current_role }} at {{ $json.current_company }}
- Industry: {{ $json.industry }}
- Headline: {{ $json.headline }}
- Experience: {{ $json.experience_summary }}
- Recent Activity: {{ $json.recent_activity }}
- Skills: {{ $json.skills }}

SCORING RUBRIC:
1. Role Relevance (0-30): Does this person make or influence database decisions?
   25-30: VP/Director/CTO with DB authority
   15-24: Senior Engineer or Lead
   5-14: Mid-level IC
   0-4: No authority (frontend, marketing, etc.)

2. DataStax/Cassandra Depth (0-30): How deeply do they use DataStax?
   25-30: Large production clusters, certified, or vocal about it
   15-24: Regular hands-on user
   5-14: Peripheral exposure
   0-4: No meaningful exposure

3. Seniority & Influence (0-20): Can they champion a technology switch?
   15-20: Budget authority or C-level
   8-14: Team lead or senior IC with influence
   0-7: IC with limited organizational influence

4. Activity Recency (0-20): Actively engaged with database pain points?
   15-20: Recent posts about DB challenges, latency, cost, scaling
   8-14: Some database-related activity
   0-7: Inactive or unrelated content

IMPORTANT: Be strict. A frontend developer should score under 10. A DataStax
employee should score under 40. A VP running a 300-node cluster with posts
about cost pain should score 85+.

RESPOND WITH ONLY THIS JSON — no markdown, no backticks, no extra text:
{"score": <0-100>, "role_relevance": <0-30>, "datastax_depth": <0-30>,
 "seniority": <0-20>, "activity_recency": <0-20>,
 "reasoning": "<2-3 sentences explaining the score>"}
```

### Why This Structure

**Why a rubric instead of just "rate this lead 0-100"?**  
Without a rubric, LLMs tend to cluster scores in the 45–65 range and struggle to distinguish a VP running 300 nodes from a junior dev who mentioned Cassandra once. The rubric forces independent evaluation of each dimension, then sums them — producing calibrated, defensible scores that you can audit.

**Why four specific criteria?**  
Each maps to a real ICP signal: role relevance (who can buy), DataStax depth (who has the pain), seniority (who can champion), activity (who is actively searching). Missing any one produces wrong results — a very senior executive who's never touched Cassandra should score lower than a mid-level engineer running 200 nodes.

**Why the "IMPORTANT: Be strict" instruction?**  
Without explicit boundary examples, LLMs over-qualify leads to seem helpful. Naming the exact edge cases ("frontend dev under 10", "DataStax employee under 40") anchors the model's calibration.

**Why include `reasoning` in the output?**  
Two reasons: (1) it makes scores auditable — a human can scan the reasoning column in Google Sheets and spot obvious errors; (2) the reasoning gets injected into both message prompts downstream, giving them context about *why* this person is valuable without re-reading the full profile.

---

## Prompt B: LinkedIn Connection Request

**Node:** `AI — Write LinkedIn Invite`  
**Model:** `claude-sonnet-4-20250514` | **Max tokens:** 300

```
You are a senior GTM professional at ScyllaDB writing a LinkedIn connection request.

ABOUT SCYLLADB:
- Drop-in Apache Cassandra replacement written in C++ — zero JVM GC pauses
- 10x better throughput at P99 latency vs DataStax
- 3-5x fewer nodes = 60-80% infrastructure cost reduction
- Used by Discord, Starbucks, Comcast, Samsung

THIS LEAD:
Name: {{ $json.first_name }} {{ $json.last_name }}
Role: {{ $json.current_role }} at {{ $json.current_company }}
Industry: {{ $json.industry }}
Experience: {{ $json.experience_summary }}
Recent Activity: {{ $json.recent_activity }}
Why they're a good prospect: {{ $json.reasoning }}

RULES — follow every one:
1. MAXIMUM 300 characters — count carefully, this is a hard limit
2. Reference ONE specific thing from their recent activity or experience
3. NO product pitch — this is a human connection, not a sales message
4. Do NOT start with "I noticed", "I came across", "I saw", or "I wanted"
5. Write as a technical peer, not a salesperson
6. Warm, direct, credible

RESPOND WITH ONLY THE MESSAGE TEXT. No quotes, no labels, no explanation.
```

### Why This Structure

**Why the 300-character hard limit?**  
LinkedIn's connection request field caps at 300 characters. The instruction must be explicit because LLMs don't know this. Setting `max_tokens: 300` adds a secondary constraint at the API level.

**Why the anti-pattern list ("do NOT start with...")?**  
"I noticed" and "I came across" are so overused in sales automation that they trigger immediate skepticism. Naming patterns to avoid is often more effective than positive instructions — the model knows these templates and needs to be explicitly told to escape them.

**Why inject the `reasoning` field from scoring?**  
The scoring node already identified the most relevant hook for this person. Passing it in saves the message prompt from re-reading the full profile and gives it a shortcut to the right angle.

**Why "NO product pitch"?**  
A LinkedIn invite is a door-opener, not a close. A message pitching ScyllaDB in 300 characters gets ignored or declined. The goal is the connection, which unlocks the follow-up email.

---

## Prompt C: Follow-up Email

**Node:** `AI — Write Follow-up Email`  
**Model:** `claude-sonnet-4-20250514` | **Max tokens:** 800

```
You are a GTM professional at ScyllaDB writing a follow-up email.
The LinkedIn connection was accepted 3 days ago.

ABOUT SCYLLADB:
- Drop-in Apache Cassandra replacement, fully CQL-compatible (zero code changes)
- Written in C++ — eliminates JVM GC pauses entirely
- 10x better throughput at P99 latency
- 3-5x fewer nodes = 60-80% infrastructure cost reduction
- Customers: Discord (messages), Starbucks (rewards), Comcast (CDN), Samsung

LEAD:
Name: {{ $json.first_name }} {{ $json.last_name }}
Role: {{ $json.current_role }} at {{ $json.current_company }}
Industry: {{ $json.industry }}
Experience: {{ $json.experience_summary }}
Recent Activity: {{ $json.recent_activity }}
Why they scored highly: {{ $json.reasoning }}
LinkedIn invite we sent: {{ $json.linkedin_invite }}

RULES — follow every one:
1. Write a compelling subject line specific to their situation (not generic)
2. Open by referencing the LinkedIn connection naturally (1 sentence)
3. Identify 1-2 specific pain points from their profile/activity — be precise
4. Position ScyllaDB as the solution to THEIR pain, not as a feature list
5. Name a relevant customer (Discord for scale, Comcast for telecom, etc.)
6. ONE soft CTA: 15-min call OR a case study — not both, not a demo request
7. Total body under 200 words
8. Tone: consultative peer, not salesperson
9. NEVER use: "hope this finds you well", "reaching out", "synergy",
   "leverage", "circle back"

RESPOND WITH ONLY THIS JSON — no markdown, no backticks:
{"subject": "<subject line>", "body": "<full email body>"}
```

### Why This Structure

**Why inject the LinkedIn invite text?**  
The follow-up email should feel like a continuation, not a cold start. Passing the invite text means Claude can reference the specific hook used and build on it, creating message continuity.

**Why customer logos in the context?**  
Social proof anchors in the prompt guide the model to choose logos relevant to the prospect's industry. A telecom DBA hears "Comcast" and immediately maps it to their own environment. The model selects the right example based on the lead's industry field.

**Why "identify pain points — be precise" rather than "explain features"?**  
Feature-led outreach ("ScyllaDB has 10x throughput!") reads like a brochure. Pain-led outreach ("You wrote about GC pauses eating your P99 budget — we eliminated that at the architecture level") reads like the sender actually understood the problem. The rule forces personalization before positioning.

**Why the explicit "NEVER use" list?**  
Same principle as the LinkedIn anti-patterns. "Hope this finds you well" is the most reliable signal that an email was written by automation, not a human. Naming it by name is more effective than "write naturally."

**Why JSON output with separate subject and body?**  
The workflow writes subject and body to separate columns in Google Sheets. A single text blob would require additional parsing. Structured output maps cleanly to the logging node.

---

## System-Level Design Decisions

### Why run scoring and message generation as separate API calls?

Combining all three in one prompt would ask the model to evaluate, score, and write messages in a single response — each task performs worse when combined. The separate calls create a "chain of thought relay": scoring produces `reasoning`, which seeds message generation, which produces messages referencing the exact right context.

### Why Claude specifically?

Claude's instruction-following on format constraints (pure JSON, character limits, banned phrases) is more reliable for structured output tasks. The `claude-sonnet-4-20250514` model is fast enough for real-time workflow use and precise enough for both scoring rubrics and tone-sensitive messaging.

---

## What Would Improve These Prompts in Production

| Improvement | Impact |
|---|---|
| **Few-shot examples** — add 2-3 good/bad invite examples to Prompt B | Tightens tone consistency significantly |
| **Industry-specific injections** — prepend telecom/fintech/ad-tech pain point lists | More precise pain-point identification |
| **Chain-of-thought scoring** — ask model to "think step by step" before JSON | ~15% improvement in rubric adherence |
| **A/B prompt variants** — two slightly different email tones, track reply rates | Data-driven optimization vs guesswork |
| **Negative example leads in scoring prompt** — "a frontend dev should score like X" | Better calibration at the tails |

---

## Considerations & Recommendations

### What this system does well
- The 4-criteria rubric prevents score inflation — leads get genuinely differentiated scores
- Chaining score reasoning into messages creates real personalization, not mail-merge fake personalization
- The mock data fallback with intentionally bad leads proves the filter actually works
- Sequential message generation (invite → email) keeps tone and reference consistent

### What to watch for in production
- **Prompt drift:** Claude updates may change output format. Add a post-processing validation step that checks for required JSON keys before proceeding.
- **Rate limits:** Anthropic free tier allows ~5 requests/min. At 10 leads, that's ~10 API calls. At 100 leads, consider batching or adding delays.
- **Score calibration:** Run the workflow on 20 known leads (mix of ideal prospects and obvious non-prospects) and verify the scores match your expectations before running it live.
- **Message quality review:** Always human-review the first 10 generated messages before approving any real outreach. LLMs occasionally hallucinate specific details ("your 500-node cluster" when they have 120 nodes) that would embarrass you.
