# X For You Tweet Generator

## Purpose

Generate high-performing X posts using the open-sourced For You feed pipeline in /workspaces/metana/x-algorithm as the decision model.

Use this skill whenever the user asks for:
- a tweet
- an X post
- a thread opener
- multiple tweet variants to choose from

The goal is to maximize positive engagement signals while minimizing negative feedback signals and filter risk.

## Grounding (from algorithm)

The feed pipeline effectively rewards content with strong predicted engagement and penalizes negative interactions:
- Positive signals include favorite, reply, repost, click/profile click, share, copy-link share, follow-author, quote, and dwell behavior.
- Negative signals include not interested, mute author, block author, report, and not-dwelled behavior.
- Freshness and eligibility matter: old, duplicate, seen, muted-keyword, blocked/muted-author content is filtered.
- Diversity matters: repeated author/topic patterns are attenuated.

When numeric weights are unavailable in-repo, treat this as a relative priority system.

## Operating Rules

1. Start from user intent and audience
- Extract topic, goal, target audience, desired tone, and call-to-action.
- If missing, infer from context and proceed (do not block unless user asks for strict constraints).

2. Optimize for weighted engagement proxies
- Prioritize clarity + curiosity in the first line.
- Include one concrete value element (number, mini-framework, checklist, example, hard claim, or strong opinion).
- Prefer prompts that invite replies (question, challenge, tradeoff, hot take with rationale).
- Use language that can trigger repost/share value (teach, summarize, or provide reusable insight).
- Encourage dwell by making each sentence carry information density (no filler).

3. Minimize negative-feedback risk
- Avoid spammy phrasing, vague hype, engagement bait, or manipulative urgency.
- Avoid likely muted-keyword clusters (generic scammy growth language, repetitive hashtags, low-value keyword stuffing).
- Avoid polarizing abuse, personal attacks, or misleading claims.

4. Respect filter-like constraints
- Keep tweet content self-contained and readable without external context.
- Do not produce near-duplicate variants; each option must differ in angle and hook.
- Prefer timely framing ("today", "this week", current shifts) when relevant.

5. Diversity across batches
- If generating multiple options, vary:
  - hook pattern (contrarian, data-led, story-led, framework-led)
  - sentence rhythm/length
  - CTA type (question, opinion prompt, experiment invitation)

## Output Format

For each request, return:

1) Best Tweet
- final tweet text only

2) Why This Should Rank
- 3 to 5 short bullets tied to engagement proxies (reply/share/dwell/click/follow) and risk reduction

3) Alternates (optional)
- 2 to 4 distinct variants if user asks for options

4) Quick Scorecard
- Reply potential: Low/Med/High
- Repost/share potential: Low/Med/High
- Dwell potential: Low/Med/High
- Negative feedback risk: Low/Med/High

## Length and Style Constraints

- Default target: <= 280 chars.
- Keep hashtags minimal (0 to 2 max, only if genuinely useful).
- Emojis optional, sparse, and purpose-driven.
- No thread unless user asks.
- No fabricated data, metrics, or citations.

## Prompting Template (internal)

Use this internal structure before producing output:

- Topic:
- Audience:
- Intent (teach/opinion/announcement/question):
- Primary engagement target (reply/repost/share/dwell/follow):
- Risk constraints (words/angles to avoid):
- Draft 3 hooks:
- Select best draft + scorecard:

## Fast Modes

Use one of these modes when user asks:
- "reply-max": optimize for comments and conversation starters.
- "share-max": optimize for repost/copy-link utility.
- "dwell-max": optimize for depth and reading completion.
- "balanced": optimize a blend of reply, share, and dwell.

## Example Invocation

User: "Write me a tweet about learning Solidity security audits."

Assistant should:
1. Draft a concise high-signal tweet.
2. Briefly explain ranking rationale using engagement proxies.
3. Provide alternates if requested.

## Notes

- Use this skill as the default behavior for tweet-generation requests in this workspace.
- If user supplies a hard style guide, obey the user style guide first, then optimize within those bounds.
