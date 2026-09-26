---
name: x-algorithm-optimizer
description: Optimize tweets and X posts for reach using the open-sourced X (Twitter) For You feed algorithm. Rewrite drafts, diagnose why a post underperformed, and explain engagement, ranking, and shadowban mechanics from the actual xai-org/x-algorithm source — candidate sourcing, filters, Phoenix scoring, reply ranking, and visibility filtering. Interprets "Under the Hood" label reports and audits an account's recent posts.
license: Apache-2.0 (references xai-org/x-algorithm)
---

# X Algorithm Optimizer

Grounded in the August 2026 source release: `github.com/xai-org/x-algorithm`.

**This supersedes the 2023 `twitter/the-algorithm` model.** Real-graph, TwHIN, Tweepcred, UTEG, tweet-mixer and the search-index candidate source are **not** in the current system and must not be cited. SimClusters is the only named component that survived.

## When to Use

- Optimizing a draft post or thread for reach
- Diagnosing why a post underperformed
- Planning a posting strategy for a niche or launch
- Explaining X distribution mechanics to someone
- "Am I shadowbanned?" / interpreting an Under the Hood JSON report
- Auditing an account's last N posts

Do not use for: LinkedIn/Medium/blog content, general copy editing, or tone work unrelated to distribution.

## Epistemic Rule (non-negotiable)

Separate three tiers in every analysis. Never blur them.

1. **In the code** — pipeline stages, filter names, constants, struct fields. State these as fact.
2. **Inferred from the code** — tactical advice that follows from a mechanism but isn't itself in the source. Label it as inference.
3. **Folklore** — "post at 9am", "the first hour decides everything", "links get suppressed". Say plainly when something is not visible in the released code.

**The trained weight values are NOT published.** `ScoringWeights` is a public struct with ~40 fields loaded via `params.get()` — no defaults in source. Anyone who tells you a like is worth "0.5" or a repost "20x" is inventing it. Never rank the signals numerically. Rank them *structurally*: which are modeled, which are penalized, which gate distribution entirely.

## How the Feed Actually Works

~500M daily posts → ~1,500 candidates → **35 shipped** (`RESULT_SIZE`), with `TOP_K_CANDIDATES_TO_SELECT = 50`.

### Stage 1 — Candidate sourcing (three parallel sources)

| Source | What it does | What it means for you |
|---|---|---|
| `thunder` | In-memory index of recent posts from accounts the viewer follows | Your followers are the only guaranteed audience |
| `phoenix` retrieval | Embeds viewer and post as vectors, returns nearest posts | Out-of-network reach is **semantic**, not keyword or hashtag |
| `simclusters` | Clusters accounts by who engages with what | Community coherence is a retrieval mechanism, not a metaphor |

There is no keyword search index feeding the For You feed. **Hashtags are not a retrieval path.** Topical clarity matters because it moves your embedding, not because it matches a string.

### Stage 2 — Pre-scoring filters (a post killed here is never ranked)

Applied in order: `DropDuplicates` → `CoreDataHydration` → `Age` → `SelfTweet` → `OONRetweetReply` → `PreviouslySeen` → `MutedKeyword` → `AuthorSocialgraph`.

Three of these are strategically load-bearing:

- **`AgeFilter` — `MAX_POST_AGE = 48 * 60 * 60`.** Hard 48-hour window. A post has two days of candidate life, full stop. Evergreen content does not accrue reach; it must be reposted as new.
- **`OONRetweetReplyFilter`** drops reposts *and replies* from accounts the viewer doesn't follow. **Replies essentially cannot reach out-of-network.** This is the single biggest structural fact for content shape.
- **`PreviouslySeenPostsFilter`** — one impression per viewer. No second bite.

### Stage 3 — Phoenix scoring

A transformer predicts probabilities for ~40 distinct actions. `Final Score = Σ (weight_i × P(action_i))`.

Modeled positives: `favorite`, `reply`, `retweet`, `quote`, `share`, `share_via_dm`, `share_via_copy_link`, `click`, `open_link`, `profile_click`, `photo_expand`, `video_open`, `vqv`, `quoted_click`, `quoted_vqv`, `dwell`, `cont_dwell_time`, `cont_click_dwell_time`, `follow_author`, `post_unexplored`.

Modeled negatives: `not_interested`, `block_author`, `mute_author`, `report`, **`not_dwelled`**.

Five mechanisms in `ranking_scorer.rs` that change how you write:

- **`share_via_dm` and `share_via_copy_link` are separately modeled.** Private sharing is a first-class ranked signal. Content worth sending to one person counts, and it never shows up in your public metrics.
- **`not_dwelled` is a negative.** A scroll-past is not neutral — it's modeled damage. This is why volume posting is genuinely costly now.
- **Click-dwell low-favorability penalty** — `multiplier = (fav / baseline).powf(alpha).max(floor).min(cap)`, damping `click_dwell_time` when favorites come in low relative to baseline. **This is an explicit anti-clickbait term.** A hook that earns the click but not the approval gets its dwell credit cut.
- **`bidirectional_follow_reply_weight_boost` / `..._dwell_weight_boost`** — mutual follows get an additive boost on the reply and dwell weights, applied to original posts. **Mutuals are a code-level multiplier.** Building genuine two-way relationships is a ranking strategy, not just etiquette.
- **`post_unexplored`** — a novelty term, optionally multiplicative: `base_dwell_time * (1.0 + post_unexplored * alpha)`. Freshness of *idea*, not just timestamp.

**Video view credit is gated** (`value_model.rs` → `candidates_util::vqv_eligible`). `vqv` only counts when the video's duration is `> min_video_duration_ms` (a param, value unpublished), and only for viewers with `< MAX_FOLLOWERS_THRESHOLD = 10_000` followers. A clip that's too short can't earn `vqv` at all, so "shorter is always better" is wrong. The real rule: pass the minimum and hold attention. Every second a stranger doesn't stay risks `not_dwelled`.

Dwell-regret scoring modes (`dwell_regret_sigmoid`, `gated_dwell_regret`) compare a post's engagement to cohort averages via a centered ratio. You are scored **against comparable posts**, not on an absolute scale.

**Candidate isolation:** during inference candidates cannot attend to each other — each post is scored against viewer context alone. Your score does not depend on what else is in the batch. There is no "competing with a viral post for the slot" at scoring time. Do not give timing advice that assumes there is.

### Side surface — reply ranking inside conversations

For You is not the only surface. Replies to a post are ordered by an LLM judge in `grox/flows/reply_spam/`. `ReplyScorer` (Grok 4 mini, with a Gemma fallback on smaller threads) reads the rendered thread with engagement signals and follower counts. It returns `ReplyScoreResult { score: float, reason: str }`, bucketed 0–3. Its prompts are **withheld** "to reduce gameability". The same folder also holds `coordinated_spam` and `multi_step_reply_spam` classifiers.

What this means (inference): a reply that stands on its own gets placed higher under the parent and borrows the parent's audience. A one-token reply ("AGI", "this", "demon time") has nothing for the judge to rate. For small accounts this is the main path to strangers, since replies can't travel via For You (`OONRetweetReplyFilter`). But it only reaches the parent's viewers, so it builds relationships and mutuals, not a broadcast.

### Stage 4 — Score adjustments

- **Author diversity decay:** `(1.0 - floor) * decay^k + floor`, where `k` counts your repeats within the ranked list. Posting many times in a short window means each subsequent post is scored lower **for the same viewer**. Real, and it caps burst posting.
- **Out-of-network discount:** a sub-1.0 factor on posts from unfollowed accounts — and per the README, **on replies and reposts even from followed creators**.
- **New-author boost:** authors under an impression threshold get an uplift toward a target position (`author_cold_start.rs`). New accounts have a real, temporary tailwind.
- **`NEW_USER_OON_WEIGHT_FACTOR = 0.00001`** (for viewers with `< NEW_USER_MIN_FOLLOWING = 5`). A new *viewer's* feed is essentially their follows only. Content aimed at brand-new users cannot arrive out-of-network.

### Stage 5 — VMRanker diversity pass

`vm_ranker` reorders scored posts with a **determinantal point process** over their embeddings, trading score for dissimilarity between neighbors. The top-scoring feed is deliberately not the shipped feed. **Being the seventh near-identical take on a trending topic is penalized by construction** — semantic differentiation is worth more than incremental quality on a crowded subject.

### Stage 6 — Visibility filtering (separate service, separate rules)

Returns **ALLOW / INTERSTITIAL / DROP** per post-viewer pair. It decides *whether*, never *where*. Post-selection filters `VFFilter`, `AncillaryVFFilter`, and `DedupConversationFilter` apply after order is fixed.

Critically: **some rules apply only to out-of-network recommendations** — high-recall spam catching for strangers while followers see the identical post. That asymmetry is the honest technical core of the "shadowban" question: not a score nudge, a label-driven rule. Per-account labels are surfaced in `under-the-hood/`.

Label sources feeding it:
- `grox` — spam, adult, violent-media classifiers at publish time
- `agatha` — offline jobs labeling an account by **how others respond to its posts: blocks, reports, spam reports**
- `bdsm` — action-sequence analysis for inauthentic/abusive behavior patterns
- `user-cred-v2` — PageRank over the follow graph and engagement edges → per-account score (the structural successor to Tweepcred; do not call it Tweepcred)

`agatha` is why engagement-through-outrage is a losing trade: the block and report rates it aggregates are account-level and persistent, and they route into a system that can gate you out-of-network entirely — where all your growth lives.

### Under the Hood — the user-facing label report

`under-the-hood/` is also a downloadable per-account JSON report (rolled out to all eligible users Sept 2026). This is the only first-party evidence for "am I shadowbanned?", so ask for it before speculating.

Shape:
```json
{ "period": { "startDate": "2026-08-01", "endDate": "2026-08-31", "timezone": "UTC" },
  "generatedAt": "2026-09-09T23:59:59Z", "postCount": "70",
  "postLabels": [], "accountLabels": [], "totalPostLabels": 0, "totalAccountLabels": 0 }
```

From `underTheHoodReport.User.strato`:
- Eligibility: `minimumAccountAge = 365.days` **and** `minimumEligiblePosts = 10L` in the prior month.
- Covers the **last completed calendar month** only, available `minDaysAfterMonthEnd = 10` days after it ends. It never describes this week.

The label catalogue (`strato/lib/underTheHoodLabels.strato`) states each label's effect in plain English. Group them by what they do to reach:

| Effect | Post labels | Account labels |
|---|---|---|
| **Hidden from recommendations to non-followers** (the real "shadowban") | `SPAM_HIGH_RECALL`, `MALICIOUS_URL`, `DO_NOT_AMPLIFY`, `FOSNR_ABUSE_INSULTS`, all `NSFW_*` / `GORE_AND_VIOLENCE_HIGH_PRECISION` (these also hide from minors and logged-out users) | `SpamHighRecall`, `DoNotAmplify`, `ImpersonationHighPrecision`, `AbusiveHighRecall`, `Compromised`, `ReadOnly`, all `Nsfw*` |
| **Profile-only** (with a visible limited-visibility notice) | `FOSNR_ABUSE`, `FOSNR_HATEFUL_CONDUCT`, `FOSNR_VIOLENT_SPEECH`, `FOSNR_CIVIC_INTEGRITY` | — |
| **Not shown at all** | `SPAM`, `PDNA` (pending review), `BOUNCE` (pending author deletion) | — |
| **Not in Home timeline** | `FOR_EMERGENCY_USE_ONLY` | — |
| **Legal / copyright withholding** | `LegalRequest(cc)`, `BystanderReport(cc)`, `UnspecifiedReason(cc)`, `Dmca` / `is_dmca` | same, per country |

**Reading a clean report (all empty, totals 0):**
- Proves: no catalogued label hit that month's posts or account. The OON visibility gate was not closed on you.
- Does not prove: good ranking. Low predicted engagement is invisible in the report and feels identical to a ban from inside analytics.
- Does not cover: the current month, or classifier rules X keeps private (the report calls itself "best-effort").

A clean report moves the diagnosis from Stage 6 to Stages 3–5. Say so directly. Don't leave room for a hidden-curse story.

## Optimization Playbook

### Content shape (from the filters)

1. **Original posts are the growth vehicle. Replies are relationship maintenance.** `OONRetweetReplyFilter` means replies effectively don't travel out-of-network. Reply-guy strategy builds mutuals (real boost) but not reach.
2. **Threads: the hook post carries the distribution.** Continuations are replies and inherit the same limitation. Front-load the whole value proposition in post one; never let it be a teaser.
3. **48 hours, then it's dead.** Plan launches accordingly. Re-post evergreen material as new posts rather than resurfacing old ones.
4. **Quotes are ranked separately** (`quote`, `quoted_click`, `quoted_vqv`) and are not subject to the reply filter the way replies are. Quoting with substantive added value is the stronger amplification move. A quote that only reacts to a viral post ("this guy is on demon time", "you guys are getting paid?") adds nothing new. It sits semantically on top of its parent (DPP) and gives a stranger no reason to stop.
5. **Replies: a full sentence into a live thread.** They get ranked by the `grox` reply judge, not by For You. Aim at mid-size threads where a good reply can still place high. Don't expect them to travel beyond that thread.
6. **Video: clear the `vqv` duration floor, then cut hard.** Put the payoff in frame 1. A long changelog video from a small account mostly produces `not_dwelled` (inference). A tiny clip below the floor forfeits `vqv` entirely (code).

### Signal shape (from the scorer)

7. **First line = the problem, never the product name.** An unfamiliar name ("JEV", "Hanami", "v0.6.0") means nothing to a stranger and gives the model nothing to embed. "Claude sits on a permission prompt for 20 minutes" earns the stop; "Companion TTS v0.6.0" doesn't (inference from `not_dwelled` + `phoenix` retrieval).
8. **One cluster per stretch.** Product, then local joke, then AI hot take, then a demo: that mix spreads your posts across unrelated topics, so neither `simclusters` nor `phoenix` retrieval knows which strangers to show you to (inference).
9. **Write for the private share.** `share_via_dm` and `share_via_copy_link` are modeled. Ask: would someone send this to one specific person? That intent ranks and never appears in your like count.
10. **Kill the scroll-past.** `not_dwelled` is a modeled penalty. First line must stop the thumb or the post is net-negative.
11. **No naked curiosity hooks.** The click-dwell low-fav penalty explicitly punishes clicks that don't convert to approval. Deliver in-post; earn the click *and* the like.
12. **Cultivate mutuals deliberately.** The bidirectional-follow boost is in the source. Fifty real mutuals in your niche beat five thousand one-way followers.
13. **Be differentiable, not just correct.** The DPP pass penalizes semantic neighbors. On a crowded topic, take the angle no one else has rather than the best version of the common one.
14. **Novelty is modeled** (`post_unexplored`). New ideas are scored, not just new timestamps.

### Restraint (from the adjustments)

15. **Space your posts.** Author diversity decay is per-viewer and per-ranked-list. Burst posting devalues your own posts against each other.
16. **Protect account-level standing.** Blocks, reports, and mutes feed `agatha` and `bdsm` at the account level and can gate out-of-network distribution. Manufactured outrage borrows reach against your entire future.
17. **New accounts: exploit the cold-start boost, and expect a follower-only ceiling if your audience is itself new.**

### Do not claim

- ❌ Numeric weights or ratios between signals — not published
- ❌ "The first hour is critical" / velocity multipliers — **no such term is visible** in the released ranking code
- ❌ Hashtag or keyword optimization — no keyword retrieval path into For You
- ❌ Link penalties — `open_link` and `click` are modeled **positives**; no link demotion appears in the source
- ❌ Optimal posting times — not in the code. Timing helps only through who is online to generate the signals.
- ❌ Real-graph, TwHIN, Tweepcred, UTEG, tweet-mixer — retired or renamed
- ❌ "Earn N out-of-network likes fast and I'll expand you" — there's no staged-expansion or velocity gate in the code. It's the first-hour myth in different words.
- ❌ "You're competing with everyone posting at the same hour" — candidates are scored in isolation. Only the DPP pass and author diversity compare posts with each other.
- ❌ "Recency penalty for repeating a story" — the real mechanisms are `PreviouslySeenPostsFilter` (per viewer), the DPP similarity pass, and author diversity decay. Name those.
- ❌ "Videos must be under ~20s" — no ceiling in code. There's a *minimum* (`min_video_duration_ms`) for `vqv` credit.
- ❌ Premium / verification reply boosts — not visible in the released ranking code. Mark as unverified.
- ❌ Grok (the chatbot) role-playing "I am the X algorithm" — it's an LLM with web search, not the ranker. Its audits mix real mechanisms with the myths above. Treat its output as a draft to check against the code.

## Working Method

**Step 1 — Classify the post.** Original / reply / quote / thread-head / repost. This alone determines the reach ceiling via the pre-scoring filters. State it first.

**Step 2 — Audit the funnel.** Does it survive Age, OONRetweetReply, and the muted/blocked filters? A post that dies in filtering can't be fixed by better copy.

**Step 3 — Map to modeled signals.** Which of the ~40 heads does this plausibly fire? Name them literally (`share_via_dm`, `profile_click`, `cont_dwell_time`). Then find the one you're leaving on the table.

**Step 4 — Check the penalties.** Scroll-past risk (`not_dwelled`)? Clickbait shape (click-dwell low-fav)? Semantic duplicate of the timeline (DPP)? Third post this hour (author diversity)? Block/report risk (`agatha`)?

**Step 5 — Rewrite, then explain each change by mechanism.** Every edit cites a real code path or gets labeled inference. No mechanism, no claim.

### Mode: "Am I shadowbanned?"

1. Ask for the Under the Hood JSON. If they're not eligible (account < 1 year old, or < 10 posts last month), say that no first-party evidence exists.
2. Labels present → map each one to its effect using the catalogue table. Say plainly which ones gate out-of-network reach.
3. All empty → not labelled **for that month**. Move the diagnosis to ranking (steps 1–5 above) and to graph size. Point out the report lags by one month and is best-effort.
4. Don't invent a hidden label to explain low reach. Low predicted engagement is the default explanation. It's not a conspiracy.

### Mode: account audit ("analyze my last N posts")

Go through each post, newest first: type (step 1) → what a stranger sees in the first line or first frame → which signal heads it plausibly fires → the mechanism that kills it. Then summarize across posts:
- **Topic coherence**: one cluster, or a mix that confuses the retrieval embedding?
- **Hook pattern**: problem-first or product-name-first? Find the account's own best posts and show that they already used the winning pattern.
- **Shape mix**: how much output is originals vs. no-take quotes vs. one-token replies?
- **Video lengths**: relative to the `vqv` floor and to what a stranger would sit through.
- **Graph size**: with a small follower base, in-network reach is small, so OON retrieval and replies in live threads carry the account.

Close with ≤6 concrete changes, each tagged code or inference.

## Worked Example

**Original:**
> "We launched a new feature today. Check it out."

**Analysis:**
- Original post — full reach path available, good.
- Survives all pre-scoring filters.
- Fires almost nothing: no dwell hook, no private-share value, no `profile_click` pull. High `not_dwelled` risk.
- "Check it out" with no in-post payload is the exact shape the click-dwell low-fav penalty is built to dampen.
- Semantically near-identical to every launch post → the DPP pass buries it next to its neighbors.

**Optimized:**
> "Six months on the one thing users asked for most: PDF export.
>
> Report generation went from 40s to 4s. Live now.
>
> The hard part wasn't the rendering — it was that our layout engine assumed infinite page height. Here's what we changed:"

**Why:**
- Concrete numbers → `cont_dwell_time` and bookmark/DM-share intent (`share_via_copy_link`), not a bare announcement.
- The specific failure detail is `post_unexplored` territory — a genuinely unexplored angle, and the thing that makes it semantically distinct under the DPP pass.
- Value delivered in-post before any ask → no click-dwell penalty exposure.
- Technical specificity moves the embedding toward the engineering cluster (`simclusters` / `phoenix` retrieval), rather than into the generic-launch neighborhood.
- Zero block/report surface — no `agatha` cost.

## Source

`github.com/xai-org/x-algorithm`, Apache-2.0, released August 2026. Model weights, exact coefficient values, and Grok-based policy-violation prediction are excluded from the release. When a question requires those, say so rather than guessing.
