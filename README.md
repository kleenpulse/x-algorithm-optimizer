# X Algorithm Optimizer

An agent skill for optimizing posts on X, grounded in the **actual open-sourced For You feed algorithm** — [`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm), released August 2026 under Apache-2.0.

Not vibes. Not 2023 folklore. Code paths.

## Install

```bash
npx skills add kleenpulse/x-algorithm-optimizer
```

Works with Claude Code, Codex, Cursor, Copilot, and anything else that reads `SKILL.md`.

## Why this one

Most X-growth advice is built on the 2023 `twitter/the-algorithm` release — or on nothing at all. **Real-graph, TwHIN, Tweepcred, UTEG, and the tweet-mixer no longer exist.** SimClusters is the only named component that survived into the current system.

This skill is rebuilt against the 2026 source, and it teaches the agent to separate three tiers and never blur them:

| Tier | Example |
| --- | --- |
| **In the code** | `MAX_POST_AGE = 48h`. Stated as fact. |
| **Inferred from the code** | "Front-load the hook" — follows from `not_dwelled` being a modeled penalty. Labeled as inference. |
| **Folklore** | "The first hour is critical." Called out as **not present** in the released code. |

**The trained weights are not published.** `ScoringWeights` is a public struct with ~40 fields, all loaded via `params.get()`, no defaults in source. So the skill is explicitly forbidden from ranking signals numerically. Anyone quoting you a "20x repost multiplier" invented it.

## What it actually knows

- **The funnel** — ~500M posts → ~1,500 candidates → **35 shipped** (`RESULT_SIZE`), via three parallel sources (`thunder` in-network, `phoenix` embedding retrieval, `simclusters`).
- **`OONRetweetReplyFilter`** — replies effectively cannot reach out-of-network. Reshapes every thread and reply strategy.
- **`MAX_POST_AGE = 48 * 60 * 60`** — hard two-day candidate lifespan. Evergreen content does not accrue.
- **`not_dwelled` is a modeled negative** — a scroll-past is damage, not neutral.
- **The anti-clickbait term** — `(fav / baseline).powf(alpha)` damps click-dwell credit when clicks don't convert to approval.
- **`bidirectional_follow_reply_weight_boost`** — mutual follows are a code-level ranking multiplier.
- **`share_via_dm` / `share_via_copy_link`** — private sharing is separately modeled and never appears in your public metrics.
- **VMRanker's determinantal point process** — semantic near-duplicates are penalized by construction. Differentiation beats incremental quality.
- **Visibility filtering** — ALLOW / INTERSTITIAL / DROP, a separate service from ranking, with rules that apply *only* to out-of-network recommendations. The honest technical answer to the shadowban question.
- **`agatha` / `bdsm` / `user-cred-v2`** — account-level labels derived from how others respond to you (blocks, reports, spam reports).

## What it refuses to claim

- Numeric weights or ratios between signals — not published
- First-hour velocity multipliers — **no such term** in the released ranking code
- Hashtag or keyword optimization — no keyword retrieval path feeds For You
- Link penalties — `open_link` and `click` are modeled **positives**
- Optimal posting times — not in the code
- Real-graph / TwHIN / Tweepcred / UTEG / tweet-mixer — retired

## Usage

Once installed, ask naturally:

- "Optimize this post for reach: `<draft>`"
- "Why did this underperform?"
- "Is a thread or a single post better for this?"
- "How does out-of-network distribution actually work?"

The skill classifies the post type first (original / reply / quote / thread-head), audits it against the pre-scoring filters, maps it to the modeled signals by name, checks the penalties, then rewrites — citing a real code path for every change.

## Source & attribution

Derived from reading [`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm) (Apache-2.0). No source code is redistributed here — this is analysis and prose. Model weights, exact coefficients, and the Grok-based policy-violation prediction are excluded from X's release; where a question depends on those, the skill says so instead of guessing.

## License

MIT — see [LICENSE](LICENSE).
