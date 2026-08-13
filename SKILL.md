---
name: x-algorithm-optimizer
description: Optimize tweets and X posts for reach using the open-sourced X (Twitter) For You feed algorithm. Rewrite drafts, diagnose why a post underperformed, and explain engagement, ranking, and shadowban mechanics from the actual xai-org/x-algorithm source — candidate sourcing, filters, Phoenix scoring, and visibility filtering.
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

Dwell-regret scoring modes (`dwell_regret_sigmoid`, `gated_dwell_regret`) compare a post's engagement to cohort averages via a centered ratio. You are scored **against comparable posts**, not on an absolute scale.

**Candidate isolation:** during inference candidates cannot attend to each other — each post is scored against viewer context alone. Your score does not depend on what else is in the batch. There is no "competing with a viral post for the slot" at scoring time. Do not give timing advice that assumes there is.

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

## Optimization Playbook

### Content shape (from the filters)

1. **Original posts are the growth vehicle. Replies are relationship maintenance.** `OONRetweetReplyFilter` means replies effectively don't travel out-of-network. Reply-guy strategy builds mutuals (real boost) but not reach.
2. **Threads: the hook post carries the distribution.** Continuations are replies and inherit the same limitation. Front-load the whole value proposition in post one; never let it be a teaser.
3. **48 hours, then it's dead.** Plan launches accordingly. Re-post evergreen material as new posts rather than resurfacing old ones.
4. **Quotes are ranked separately** (`quote`, `quoted_click`, `quoted_vqv`) and are not subject to the reply filter the way replies are. Quoting with substantive added value is the stronger amplification move.

### Signal shape (from the scorer)

5. **Write for the private share.** `share_via_dm` and `share_via_copy_link` are modeled. Ask: would someone send this to one specific person? That intent ranks and never appears in your like count.
6. **Kill the scroll-past.** `not_dwelled` is a modeled penalty. First line must stop the thumb or the post is net-negative.
7. **No naked curiosity hooks.** The click-dwell low-fav penalty explicitly punishes clicks that don't convert to approval. Deliver in-post; earn the click *and* the like.
8. **Cultivate mutuals deliberately.** The bidirectional-follow boost is in the source. Fifty real mutuals in your niche beat five thousand one-way followers.
9. **Be differentiable, not just correct.** The DPP pass penalizes semantic neighbors. On a crowded topic, take the angle no one else has rather than the best version of the common one.
10. **Novelty is modeled** (`post_unexplored`). New ideas are scored, not just new timestamps.

### Restraint (from the adjustments)

11. **Space your posts.** Author diversity decay is per-viewer and per-ranked-list. Burst posting devalues your own posts against each other.
12. **Protect account-level standing.** Blocks, reports, and mutes feed `agatha` and `bdsm` at the account level and can gate out-of-network distribution. Manufactured outrage borrows reach against your entire future.
13. **New accounts: exploit the cold-start boost, and expect a follower-only ceiling if your audience is itself new.**

### Do not claim

- ❌ Numeric weights or ratios between signals — not published
- ❌ "The first hour is critical" / velocity multipliers — **no such term is visible** in the released ranking code
- ❌ Hashtag or keyword optimization — no keyword retrieval path into For You
- ❌ Link penalties — `open_link` and `click` are modeled **positives**; no link demotion appears in the source
- ❌ Optimal posting times — not in the code. Timing helps only through who is online to generate the signals.
- ❌ Real-graph, TwHIN, Tweepcred, UTEG, tweet-mixer — retired or renamed

## Working Method

**Step 1 — Classify the post.** Original / reply / quote / thread-head / repost. This alone determines the reach ceiling via the pre-scoring filters. State it first.

**Step 2 — Audit the funnel.** Does it survive Age, OONRetweetReply, and the muted/blocked filters? A post that dies in filtering can't be fixed by better copy.

**Step 3 — Map to modeled signals.** Which of the ~40 heads does this plausibly fire? Name them literally (`share_via_dm`, `profile_click`, `cont_dwell_time`). Then find the one you're leaving on the table.

**Step 4 — Check the penalties.** Scroll-past risk (`not_dwelled`)? Clickbait shape (click-dwell low-fav)? Semantic duplicate of the timeline (DPP)? Third post this hour (author diversity)? Block/report risk (`agatha`)?

**Step 5 — Rewrite, then explain each change by mechanism.** Every edit cites a real code path or gets labeled inference. No mechanism, no claim.

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
