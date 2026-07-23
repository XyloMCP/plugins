---
name: xylo-ads
description: Manage and analyze Meta, Google, TikTok, and X ad campaigns plus approved TikTok Shop workflows through the Xylo MCP server. Use this skill any time the user asks about ad or TikTok Shop performance, campaign management, creative, affiliates, product listings, tracking, publishing, or cross-platform analysis.
---

# Xylo Ads Skill

This is the agent playbook for the Xylo MCP server. Xylo carries **300+ operations across Meta Ads, Google Ads, TikTok Ads, X Ads, and approved TikTok Shop workflows through 28 tools**. Treat this document as the source of truth for how to use them.

**Your actual tool list is the ground truth.** The documented surface is 28 tools (21 dispatchers + `describe` + `search_tools` + 4 named AI tools + `send_feedback`). If you see hundreds of platform-prefixed tools instead (`meta_list_campaigns`, …), the user is on an old connection — everything still works, but suggest reconnecting to `https://xylomcp.com/api/mcp` when convenient.

**Staying up to date.** When any tool result begins with a block titled **"⚠️ IMPORTANT: XYLO UPDATE AVAILABLE"**, stop and relay it to the user as a highlighted callout before continuing. Codex plugin users should refresh Xylo under Settings → Plugins and start a new task; standalone-skill users should re-download the linked skill. If the notice also requests a connector refresh, include that step. Don't bury or skip it.

## How The Tools Work (read this first)

Most Xylo tools are **dispatchers**: one tool carries many routes. You pick a route with **top-level discriminators** — `channel` (`meta` / `google` / `tiktok` / `cross`), `resource`, `action`, `mode`, `lens`, `topic` — and nest **every route argument under the single `params` object**. Never put route arguments at the top level.

```
query({ channel:"meta", resource:"campaign", mode:"list",
        params:{ ad_account_id:"act_…", status:"ACTIVE", name_contains:["Q3"] } })
```

- **Reads:** `accounts` (account discovery + the saved brief), `query` (every entity read), `insights` (every metric/report), `plan` (targeting search + estimates + ad preview), `creative` (AI creative analysis/diagnosis), `competitors` (public ad libraries), `warehouse` (SQL), `knowledge` (reference brains), `klaviyo` (connected Klaviyo account: campaigns, flows, segments, profiles, metrics, attributed revenue reports), `x_ads` (X Ads accounts, campaigns, ad groups, promoted tweets, targeting, audiences, analytics, and measurement).
- **Owned channel (email/SMS):** the `klaviyo` dispatcher reads a connected Klaviyo account — campaigns, flows, segments, profiles, metrics, events, and attributed revenue reports — no ad account needed. Gated writes: create campaigns/templates/lists/segments/profiles/coupons/images; consent (single free, bulk confirmed, unsuppress one-at-a-time); flow pause/activate; `campaign.send` always dry-runs with a live recipient estimate + `confirm_token` — nothing sends without the confirm round trip; creates land as drafts.
- **Paid ↔ owned workflow:** turn a winning ad's audience into a matching Klaviyo segment for a follow-up campaign (`klaviyo({resource:"segment", action:"create"})`), or take creative-generation HTML straight into email — `klaviyo({resource:"template", action:"create"})` → `klaviyo({resource:"campaign", action:"assign_template"})`. To reuse or tweak an existing template, search by name instead of listing everything: `klaviyo({resource:"template", action:"list", params:{q:"welcome"}})` (list rows are lean — fetch the html with `template.get`).
- **Writes:** `create`, `update`, `status` (pause/resume), `delete`, `duplicate`, `media` (uploads), `publish` (organic posts + comments + TCM), `tracking` (CAPI/events), `automation` (ad rules + value rules + budget schedules), `connect`.
- **TikTok Shop:** `commerce` handles the restricted review surface: shop analytics, product listing reads, creator discovery, affiliate collaborations/free-sample settings, and creator text messages.
- **Named AI tools** (not dispatchers): `audit_campaign`, `optimize_budget`, `generate_report`, `morpheus_audit`. Plus `send_feedback`.
- **`describe`** — no args → capability overview; `{tool:"query"}` → that dispatcher's route list; `{tool, channel, resource, …}` → one route's exact schema + a valid example; batch several via `routes:[…]`. Call it before an unfamiliar **write**; reads are usually guessable.
- **`search_tools({query})`** — plain-language search over every operation; returns which dispatcher + discriminators to call.
- **Errors self-correct.** A wrong call returns the route's compact schema, a valid example, and misplaced-field hints — fix and retry once; don't spiral. An invalid combo returns the valid value list.
- **Confirm tokens.** Deletes (always), budget/bid **raises**, and large bulk activations return a `DRY RUN` preview + `confirm_token` instead of executing. Re-send the **exact same call** with `confirm_token` to execute (10-min validity). Any write also accepts top-level `dry_run:true` for a preview. This server-side gate does **not** replace asking the user (Non-Negotiables) — get their yes, then confirm.

## How To Use This Playbook

Every task runs through **The Universal Workflow** (below). Use this router to jump to the right section once the account is resolved:

| The user wants to…                                              | Go to                          | Lead call |
| --------------------------------------------------------------- | ------------------------------ | ---------- |
| See performance / ROAS / totals / top performers                | Performance Analysis           | `insights({channel:"meta", lens:"campaign"/"top_performers"})` |
| Understand *why* a metric moved ("why is CPM/CPA up?")          | Standing Lens + Knowledge      | `knowledge({topic:"meta_ads"})` first, then insights |
| Find which copy/headline works                                  | Copy & Headline Analysis       | `insights({channel:"meta", lens:"performance_by_copy"})` |
| Pause underperformers                                           | Performance Analysis → Bulk    | insights (`promoted_object` + `delivery_status`) → `update({resource:"ad", mode:"bulk"})` with `dry_run` |
| Launch / edit / duplicate / scale a campaign                    | META — Create                  | `create({channel:"meta", resource:"campaign"→"adset"→"ad"})` |
| Choose an ad shape (single vs multi-asset vs carousel)          | Multi-Media vs Flexible        | `create` ad with `params.format` |
| Upload / group / finalize creative                              | Creative Pipeline              | `media({channel:"meta", action:…})` staging → `action:"finalize"` |
| Automate ("auto-pause", "scale budget", "daypart", "alert me")  | META — Automate                | `automation({channel:"meta", resource:"ad_rule"})` |
| Bid more for high-value audiences                               | Value Rules                    | `automation({channel:"meta", resource:"value_rule_set"})` |
| Audit account structure                                         | AI & Audit Tools               | `morpheus_audit` (deep), `audit_campaign` (quick) |
| Know *why* a creative works (hook, psychology, awareness)       | Creative Intelligence          | `creative({action:"analyze", channel:"meta"})` |
| Know *why* an ad is failing/winning (delivery + creative)       | Creative Diagnostic            | `creative({action:"diagnose", channel:"meta"})` |
| Write new creative / rewrite a hook / script a UGC ad           | Creative Frameworks            | `knowledge({topic:"creative_frameworks"})` |
| Research a competitor's ads                                     | Competitor Intelligence        | `competitors({channel, action:"brand_ads"/"search"})` |
| Look up an IG handle / monitor UGC & tagged posts               | Instagram Research             | `query({channel:"meta", resource:"instagram_account"/"instagram_media"})` |
| Publish or manage Page / IG posts & comments                    | Organic Social                 | `publish({channel:"meta", resource:"page_post"/"instagram_media"/"comment"})` |
| Google Ads work                                                 | Google Ads                     | `channel:"google"` on any dispatcher |
| TikTok Ads work                                                 | TikTok Ads                     | `channel:"tiktok"` on any dispatcher |
| TikTok Shop analytics, listings, or affiliates                  | TikTok Shop                    | `commerce({resource:"shop", action:"list"})` first |
| GMV Max (TikTok Shop automated campaigns)                       | GMV Max                        | `insights({channel:"tiktok", lens:"gmv_max"})` (the ONLY perf path) |
| X Ads work                                                      | X Ads                          | `x_ads({resource, action, params})` |
| Compare platforms                                               | AI & Audit Tools               | `insights({channel:"cross", lens:"cross_platform"})` |
| Build a live, shareable reporting dashboard (Brand+ only)       | Reporting Dashboards           | `create({channel:"cross", resource:"dashboard"})` |
| Audiences / lookalikes / customer match                         | Audiences                      | `create`/`update` audience resources per platform |
| Pixel / CAPI / offline conversions                              | Pixel & Conversions            | `tracking({channel, action:…})` |
| Onboard fresh accounts / group accounts into brands / "learn my account" | Setting Up a New Account | `knowledge({topic:"account_research"})` |
| Generate new ad creative with AI                                 | Generating Creative (AI)       | `media({channel:"meta", action:"generate"})` |
| Generate ad VIDEO with AI                                        | Generating Creative (AI)       | `media({action:"generate_video"})` (kling-video / seedance-video / omni-video; async — poll `media({action:"video_status"})`). Edit any video with natural language → `media({action:"edit_video"})`. Read `knowledge({topic:"video_generation"})` first: model routing, per-second credits, and the stills-first precision workflow (generate the exact frame as an image, then animate it via `first_frame_asset_id`) |

## Non-Negotiables

These override everything below. Never weaken them.

1. **PAUSED by default, always.** Every create lands PAUSED / DISABLE. Never silently flip to ACTIVE — tell the user it's paused and let them resume.
2. **Confirm money changes with the user.** Any budget edit, bid-strategy switch, or "apply this recommendation" needs explicit user approval first. Same for enabling any rule that spends or pauses. The server's confirm_token gate on raises/deletes is a second net, not a substitute for asking.
3. **`dry_run: true` before destructive bulk filters.** When a filter could match many entities and the change is `PAUSED` / `DELETED` / a budget cut, preview the match set first (bulk routes take `params.dry_run`; any write also takes top-level `dry_run`).
4. **Never blind-retry a failed create.** A failure usually means a validation problem — read the error (it embeds the schema + example), fix the call, then retry. (Bulk create/duplicate are the exception: they are idempotent and safe to re-run; see Bulk Operations.)
5. **Judge on the ad set's optimized event (`promoted_object`), not proxy metrics.** Read `optimization_goal` + `promoted_object` on every ad set before ranking, pausing, restructuring, or reallocating. If it names a custom event or Custom Conversion, score on that event, and carry the `promoted_object` into any replacement ad set. (Details in Performance Analysis.)
6. **`instagram_actor_id` on every Meta ad with IG placements** (from the account's `default_instagram_id`). Without it the ad only runs on Facebook.
7. **Never sum raw `action_type` rows from Meta** — use the deduped top-level fields (Performance Analysis).
8. **Warehouse tools need the data-warehousing add-on.** `warehouse({action:"query"/"schema"})` and `insights({channel:"meta", lens:"ad_integrity"})` require the add-on **and** the account sync-enabled (`warehouse_enabled: true`). Insights jobs (`lens:"job"`) need the add-on but not per-account sync. If a warehouse call returns `{ available: false }`, pivot to live tools — don't retry.

## Connector

One connector: **`https://xylomcp.com/api/mcp`** — the complete surface (all platforms, all 28 documented tools). Meta, Google, and TikTok Ads routes use the `channel` parameter; X and TikTok Shop use the resource-oriented `x_ads` and `commerce` dispatchers. Older platform-scoped connections and the previous per-tool surface keep working during the retirement window, but new setups should always use this URL.

## TikTok Shop (review org only)

Start every Shop task with `commerce({resource:"shop", action:"list"})`, then pass its `shop_id` to later calls.

- Authorization is fail-closed to exactly six seller scopes: `seller.affiliate_messages.write`, `seller.affiliate_collaboration.write`, `seller.product.basic`, `seller.creator_marketplace.read`, `seller.affiliate_collaboration.read`, and `data.shop_analytics.public.read`.
- Analytics: `performance` with `action:"shop"`, `"products"` (`mode:"list"/"get"`), `"skus"`, `"videos"`, or `"live"`. Date ranges use inclusive `start_date_ge` and exclusive `end_date_lt`.
- Listings are read-only: `product` with `action:"search"` or `"get"`; category/brand reference reads are also available. TikTok requires the unapproved `seller.product.write` scope for listing, status, price, and image mutations, so those actions are not exposed.
- Affiliates: creator search, collaboration list/create (including target-collaboration free-sample settings), and conversation list/read/send/mark-read.
- Deliberately unavailable: customer/order records, fulfillment, logistics, returns, finance, promotions, raw affiliate orders, all product writes, inventory writes, and image DMs.

## The Universal Workflow

Every task follows the same five steps. Do not skip them.

1. **Resolve the account.** If the user named a brand, call `accounts({channel:"meta"/"google"/"tiktok", action:"list", params:{query:"<brand>"}})` to resolve it to an `ad_account_id` / `customer_id` / `advertiser_id`, or `x_ads({resource:"account", action:"list", params:{q:"<brand>"}})` to resolve an X Ads `account`. Never guess. Omitting the query lists every connected account for that platform. The Meta list also returns `default_page_id` (→ `page_id`), `default_instagram_id` (→ `instagram_actor_id`), `default_catalog_id` (→ `product_catalog_id`), `warehouse_enabled` + `last_synced_at`, and the org `plan` in response meta. If the query returns multiple accounts, ask which one — don't assume.
2. **Recall the account's memory (Meta, Google, and TikTok).** Once resolved, call `accounts({action:"read_brief", params:{platform, account_id}})` BEFORE planning or writing anything. It returns the freeform `brief` and slug-keyed `memory` facts — naming conventions, default budgets, geo/audience preferences, product sets, brand voice, where new ads go. On multi-account brands it also returns `brand.accounts_knowledge`, the sibling accounts' briefs and memories, so check there before asking for context the brand already taught another account. Honor them; don't make the user repeat context. X Ads does not use the account-brief or brand-profile routes yet; work from the user's current brief and live X account resources.
3. **Pick the data source (Meta accounts only — Google, TikTok, and X have no warehouse; skip to step 4).** Check `warehouse_enabled` from the accounts list. If `true`, prefer `warehouse({action:"query"})` and the `ad_integrity` lens for slicing-heavy tasks (top N by spend, 90-day trends, every ad linking to a domain). If `false`, use live insights. Free-tier accounts are always live-only.
4. **Execute.** Run the platform sequence below. Pause for explicit approval before any write that touches money.
5. **Summarize — and remember.** Report what changed, surface IDs, flag partial failures, and frame recommendations as testable hypotheses. Whenever the user revealed a durable preference or fact, save it with `update({resource:"account_brief", params:{platform, account_id, remember:[…]}})` (one fact per `remember` entry, short stable `lowercase_snake_case` key like `naming_convention`, `default_daily_budget`, `primary_geo`, `brand_voice`, `where_new_ads_go`; re-using a key overwrites; `forget[]` drops stale keys). You don't need permission to save a fact the user clearly stated. Save durable conventions, not live state — the current budget of an existing campaign, what's active today, or a running promotion drifts; read those fresh each session and re-verify stored offers with the user before reusing them in new ads. Never store secrets or personal data. The user can view/edit everything under **Remembered by AI** in the dashboard.

### Hand the user a link into Xylo

You're the user's window into Xylo — most of them will never open the dashboard unless you point them to the exact spot. So whenever you reference a place in Xylo, or you save/create something they'd want to see with their own eyes (a brand profile you just built, a generated image, remembered facts), give them a **direct clickable link**, not a vague "check the dashboard." Treat it like a one-tap viewport: link, don't describe. All URLs are under `https://xylomcp.com`:

- **A brand** — profile (colors/fonts/voice), reference assets, saved winning ads, brand knowledge, **and every AI-generated image** all live here: `https://xylomcp.com/dashboard/brands?brand=<brand_id>`. The `brand_id` comes from `accounts({action:"read_brief"})` (`brand.id`), `accounts({action:"list_brands"})`, or the response of `update({resource:"brand"})`. This is *the* link to send after onboarding a brand ("Here's your brand in Xylo →") and after generating creative ("Open it here →").
- **Remembered by AI / an account's brief + facts** — `https://xylomcp.com/dashboard` (Ad Accounts → open the account → **Brief & Memory** tab). There's no per-account query param; name the account so they know which row to click.
- **Generation credits / billing** — `https://xylomcp.com/dashboard/billing`.
- **Connecting or managing accounts, first-time setup** — `https://xylomcp.com/dashboard/get-started`.

## Setting Up a New Account

Run when accounts are freshly connected, or whenever the user asks Xylo to "learn", "set up", or "get up to speed on" their accounts. The goal: after this runs, "put up 10 new ads in this account" needs zero follow-up questions — you already know where ads go, how they're named, what a new ad set's budget should be, where ads link, and what copy and creative win here. Two rules govern the run: **review before write** — everything inferred (brand profile, memory facts, brief) is an assumption until the user confirms it, so draft it, lay the assumptions out for the user to confirm or correct, and only then save; and **active entities first** — ingest from currently active campaigns/ad sets/ads, and if that's enough, stop (rewind in 30-day windows only when it's thin). The full protocol (including the exact memory keys to write) is `knowledge({topic:"account_research"})`; summary:

**Step 1 — group accounts into brands (whole org, one shot).** Grouping applies to Meta, Google, and TikTok accounts; X Ads does not use brand grouping yet. Even a 30-account org fits in one conversation. `accounts({action:"list"})` on every supported grouping channel + `accounts({action:"list_brands"})` (existing brands + unassigned accounts), propose the account→brand grouping to the user (infer from account names, domains, pages), confirm it, then `update({resource:"brand"})` with the full assignment list. `brand_name` finds-or-creates, so the same name never duplicates a brand; a brand's Meta, Google, and TikTok accounts attach to one brand and share one profile + asset library. Check `billing_status` in the list output: inactive accounts still get grouped, but skip them during ingestion (their query calls 403) and tell the user which need activating.

**Step 2 — ingest ONE brand per conversation.** Deep ingestion is heavy; never attempt 10+ brands in one session. Pick the user's most important brand, run the protocol below, then name the brands that remain — the user starts a fresh conversation per brand. `list_brands` (`has_profile`) plus `read_brief` per account (empty memory = not yet ingested) show where to resume.

Per brand:

1. **Research the brand website first** — with your own tooling, not a Xylo route. Brand context (products, offers, positioning) makes the account structure legible. Draft the profile (colors, fonts, `anchor_line`, voice, product truths, proof points, banned-claim risks), then **present it to the user as your assumptions and let them confirm or correct BEFORE saving** — the website is one lens; the user knows what's actually on-brand. Then save: `update({resource:"brand_profile"})` (`urls` — save the website URL itself, labeled "website") and `media({action:"save_brand_asset"})` for the logo and must-be-exact product imagery. **Colors and fonts are required, not optional** — creative generation is blind without them. Pull real values (hex codes from the site's CSS/buttons/logo, font families from CSS). **If your tooling can't read the site** (JS-rendered, blocked, or you only get markup without styles) — don't leave the fields empty and don't guess. Be direct and proactive: ask the user to send a screenshot of their homepage (*"Send me a screenshot of your site and I'll pull the exact colors and fonts off it"*). Then read the hex colors straight off the image and name the fonts (confirm the font's name with the user if the screenshot alone can't pin it down). If the user shares a public URL for that screenshot, also save it with `media({action:"save_brand_asset", params:{kind:"screenshot", url, label}})`. Only fall back to asking the user to type values by hand if they can't share an image at all.

Then for each ad account attached to the brand:

2. `read_brief` — see what's already known; don't redo it. On a brand with other researched accounts, `brand.accounts_knowledge` carries each sibling account's brief and memory: start from that and research only the delta (platform-specific conventions, budgets, structure).
3. Scan the account structure — **active entities only**: active campaigns, then ad sets of those campaigns (ad-set listing takes a `campaign_id`; no account-wide mode), a few recent active ads in full; paused/archived structure is history, not convention. Draft the operational conventions as memory facts and hold them for the step-7 review: `naming_convention`, `default_daily_budget` + `bid_strategy`, `where_new_ads_go` (the campaign/ad set new ads land in, or the settings to clone), `landing_urls` + `utm_pattern`, `placements`, `primary_geo`, audience habits. **Conventions, not snapshots**: save the typical budget for a NEW ad set, never the current budgets of existing campaigns — live state drifts, so read it fresh each session; any saved promotion/offer must be re-verified with the user before it's used in a new ad.
4. **Build the winning-ads library.** For a paid Meta account, call `media({action:"sync_winning_ads", params:{platform:"meta", account_id}})` once. The server deterministically selects and imports the winners: it groups by optimization goal, ranks on spend, rejects any false winner with zero results on the optimized event, keeps an image/video mix with an image bias, and rewinds through non-overlapping 30-day windows up to 120 days when the account is thin. Multi-media ads are credited only to an asset with positive results. This same refresh runs automatically every week; a repeat on-demand call inside six hours returns the current library without doing the work again, and a creative that already has analysis is never analyzed again. User-deleted entries stay deleted. For a Free account, or for Google/TikTok, use the manual flow: `insights({channel:"meta", lens:"winners"})` (or equivalent ad-level insights), compare ads only within one optimization goal, rank primarily on spend, reject zero-result ads, rewind one 30-day window at a time when thin, and pick 5–15 winners with an image/video mix and image bias. Save Free-plan Meta winners individually in step 6. X does not currently support the winning-ads library or AI creative analysis; use its insights for performance work without claiming creative-level analysis. Also profile the delivered audience once — `insights({lens:"account", params:{breakdowns:"age,gender"}})`, reading spend (who's served) AND results (who buys) separately — and save the dominant converting demographic as `audience_profile`.
5. `insights({channel:"meta", lens:"performance_by_copy"})` over the winners, then **synthesize — never store ad copy verbatim**. Draft a copywriting knowledge base: which angles, hooks, structures, offer framings, and CTAs convert, why they work, what to lead with next (`copy_playbook`). The test: could you write a new on-brand ad from these insights alone? Hold for the step-7 review.
6. For a paid Meta account, call `media({action:"list_winning_ads", params:{platform:"meta", account_id, detail:"full"}})` after the sync, then diff the returned stored analyses and draft the shared winning DNA (`creative_winning_dna`) without re-running `creative({action:"analyze"})`. For the Free-plan manual flow, analyze the selected winners, diff the structured fields, and call `media({action:"save_winning_ad", params:{platform, account_id, ad_id}})` on each one. **Multi-media ads in the manual flow** (one ad carrying ≥2 uploaded videos/images — `creative_format:"mixed"` or a media roster of ≥2) must be broken down first with `insights({channel:"meta", lens:"ad", params:{ad_id, breakdowns:"video_asset"}})` or `"image_asset"`; save only winning assets with their `media_index` and per-asset `metrics_snapshot`. Google and TikTok have no automatic library sync, so keep their winner research manual.
7. **Review with the user, then save.** Lay out everything you're about to write as explicit assumptions, grouped and skimmable — conventions, copy playbook, creative DNA, and the draft narrative brief — flagging what you're least sure of, and ask the user to confirm, correct, or add context only they'd know (an account shows what happened, not always what's intended). Then write it all: `update({resource:"account_brief"})` with the `remember` facts + the `brief` seed (written only while the account has no brief yet; once set it's user-owned and later updates go into `remember` facts — you get a warning instead of an overwrite).
8. Completeness check: could you answer, without asking the user — where do new ads go, how are they named, what budget and bidding, where do they link, which placements/geo/identities, what copy angles and creative style win? And: does the brand profile have concrete hex `colors`, `fonts`, and a saved logo, and does the account have a brief? Any gap, dig it out of the account or the website and save it.

Persist **conclusions**, not raw data: memory facts are synthesized insights on how this account operates and what converts in it; the brand profile captures what to generate for the brand as a whole (shared across every account attached to it); the brief is the narrative summary. Once this runs, `read_brief` returns brief, facts, the attached `brand`, brand profile, and reference assets on every future call, so the account "just works" without the user repeating themself. **Close by handing the user their brand link** — `https://xylomcp.com/dashboard/brands?brand=<brand_id>` — so they can see and tweak everything you just built (see "Hand the user a link into Xylo" above).

---

# META ADS

## The Standing Lens

These principles govern **every** Meta analysis, recommendation, and creative call — apply them before quoting topic-specific detail. The full essays live in `knowledge({topic:"meta_ads"})` (section `strategy_lens`); the reflexive decision table is here:

| Situation                                | Wrong move                     | Right move                                                              |
| ---------------------------------------- | ------------------------------ | ----------------------------------------------------------------------- |
| One bad day, panic mode                  | Pause everything               | Pull 7-day, then 30-day. Never pause on one day's data.                |
| User wants to scale                      | Narrow targeting more          | Recommend broader audience + better creative before higher budget      |
| New campaign just launched               | Obsess over learning phase     | Launch and let it run on a 7-day read                                  |
| Meta suggests an optimization            | Apply it immediately           | Question first, test small, never auto-apply                           |
| Want to beat the top ad                  | More variations of the winner  | Test **net new concepts** to scale; variations explore, rarely beat    |
| High CPM but ROAS is strong              | "Fix" the CPM                  | Leave it — marginal efficiency, not a problem (breakdown effect)       |
| Reported conversion data looks off       | Trust Meta's CPA               | Meta's ROAS is post-iOS-14 directional; cross-check attribution        |
| Account or ad got rejected/banned        | Panic, rebuild                 | May be a wave, not your ad — consider waiting it out                   |
| Conversation drifting to ad-set settings | Tune more targeting knobs      | Redirect to creative — that's where targeting actually happens         |

**Load-bearing framing:**
- **The Total Value equation governs delivery.** Meta scores each auction as `Total Value = (Bid × Estimated Action Rate) + Ad Quality`, where `Estimated Action Rate = Estimated CTR × Estimated Click-to-Conversion Rate`. Every recommendation should map back to a lever here — "improve CPA" means moving estimated CTR, CVR, or User Value, **not** "raise the bid."
- **Creative IS targeting (post-Andromeda)** — hook, format, on-screen demographics, copy, thumbnail, and landing page are now the dominant targeting input; run broad and let creative pick the audience. The only universally safe exclusion is past purchasers.

Call `knowledge({topic:"meta_ads", params:{topics:["strategy_lens"]}})` when a user asks "should I narrow/scale my audience," reacts to a one-day swing, or reports a rejection wave — and with the topic-specific packs (below) for any "why" question about delivery, cost, learning, or relevance.

### Knowledge — call it before you diagnose

`knowledge({topic:"meta_ads"})` returns Xylo-curated Meta reference (not user data). **Fire it reflexively — before pulling insights — the moment the conversation hits a "why" or "should I" question about Meta.** Don't summarize from memory; the answers depend on Meta's current behavior and Xylo's interpretation. It is cheap and prevents confident-but-wrong explanations. Also call it **before** `audit_campaign` / `morpheus_audit` — both assume you already have this context.

| User says something like…                                                                 | Pass `params.topics` |
| ----------------------------------------------------------------------------------------- | ------------- |
| "Why is CPM high / ROAS dropping / CPA up overnight / spend uneven?"                       | `auction`, `pacing` |
| "Should I scale / cut the loser? Why won't Meta spend my budget?"                          | `pacing`, `bid_strategies` |
| "It's still learning" / "why is performance so jumpy / day-to-day swings?"                 | `learning_phase`, `performance_fluctuations` |
| "Cost cap / bid cap / lowest cost / target ROAS?"                                          | `bid_strategies` |
| "Ad sets competing with each other / audience overlap / duplicate audiences"              | `auction_overlap` |
| "Quality/engagement/conversion ranking is low / relevance score?"                         | `relevance_diagnostics` |
| "Should I narrow/scale my audience?" / rejection wave / net-new vs variations             | `strategy_lens` |
| "Incrementality / true impact / which ads actually drive sales?"                          | `incremental_attribution` |
| Any other "why" about Meta delivery/cost/volume                                           | (no `topics` = full pack) |

The breakdown effect (high CPM + strong ROAS = fine, because Meta optimizes cost of the *next* conversion, not the average) is the single most common false alarm — quote it from `auction` when you see it. Learning-phase status changes which findings are even meaningful: a 40% CPA swing on an ad set still in learning is noise.

**Warehouse schema** — call `warehouse({action:"schema"})` once per session before any `warehouse({action:"query"})` (schema isn't stable across releases; never guess table/column names). Sequence: confirm `warehouse_enabled: true` → schema → compose the SQL. If either returns `{ available: false }`, pivot to live tools.

### Incremental attribution

When a user cares about **incrementality** (efficient spend, true bottom-line impact, "which ads actually drive the business"), recommend switching the ad set's attribution model from Standard to **Incremental attribution** — it optimizes for conversions the ad *caused*, stripping out ones that would have happened anyway (standard 7-day-click/1-day-view over-reports). There is **no Xylo write for this**: direct the user to Ads Manager (ad set → Conversion → Attribution setting). Works with Sales / Engagement / Leads objectives, across Web / Web+App / Web+In-Store, compatible with value optimization (since April 2025). Prefer the incremental column when comparing creatives. Frame it as a test; present the ~24% lift Meta cites as a direction, never a guarantee. Full detail: `knowledge({topic:"meta_ads", params:{topics:["incremental_attribution"]}})`.

## Performance Analysis

Default windows: **7 days for tactical (kill/scale) reads, 30 days for structural reads.** Never decide on a single day. If the range includes today, note that today's row is partial.

**Use top-level metric fields, never sum the raw `actions[]` array.** Meta returns Pixel and onsite copies of the same event; the top-level `purchases`, `leads`, `add_to_cart`, etc. are already deduped to match Ads Manager. Summing raw `action_type` rows double-counts.

### The optimized event: read `promoted_object` before judging anything

**Every analysis, ranking, restructuring, or budget decision starts by reading the ad sets' `optimization_goal` + `promoted_object`** (both returned by default on `query({channel:"meta", resource:"adset", mode:"list"/"get"})`). `promoted_object` names what the ad set actually optimizes for:

- **Standard event** — `custom_event_type: "PURCHASE" | "LEAD" | ...` → the deduped top-level fields (`purchases`, `leads`, …) are the right scoreboard.
- **Custom pixel event** — `custom_event_type: "OTHER"` + `custom_event_str: "<event>"`.
- **Custom Conversion** — `custom_conversion_id`.

For the last two, **the top-level `leads`/`purchases` fields do NOT count the optimized event.** Pull insights with `params.include_actions: true` and count the `offsite_conversion.custom.<custom_conversion_id>` rows in `actions[]`. Rank, pick winners, pause losers, and allocate budget on **that** count and its cost-per. `optimization_goal: OFFSITE_CONVERSIONS` says *that* it optimizes conversions; only `promoted_object` says *which* one. This also applies to the `top_performers` lens, `audit_campaign`, and `optimize_budget` — their rankable metrics count standard events, so re-check any ad set whose `promoted_object` points at a custom event before acting.

**Before diagnosing underperformance,** call `knowledge` (above). Don't invent explanations. For structural audits use `morpheus_audit` (deepest — architecture, retargeting, creative fatigue, overlap, tracking health); for quick benchmark scoring use `audit_campaign`.

**Lens selection** (all `insights({channel:"meta", lens:…, params:{…}})` unless noted):

| Question                                               | Lens / call                                              |
| ------------------------------------------------------ | ----------------------------------------------------- |
| Account-level totals                                   | `lens:"account"`                                       |
| Per-campaign / per-adset / per-ad                      | `lens:"campaign"` / `"adset"` / `"ad"`                 |
| Top N by metric                                        | `lens:"top_performers"`                                |
| Side-by-side campaigns                                 | `lens:"campaign"` with `params.campaign_ids`           |
| Which copy/headline works                              | `lens:"performance_by_copy"`                           |
| Which products work (catalog ads)                      | any lens with `params.breakdowns:"product_id"` → `query({resource:"catalog_product", mode:"list"})` |
| Heavy: 90d+ ad-level, big accounts, breakdowns         | `lens:"job", mode:"job_create"` → `mode:"job_check"` (needs the warehousing add-on) |
| Cross-platform totals                                  | `channel:"cross", lens:"cross_platform"`               |
| Slice-and-dice on warehouse accounts                   | `warehouse({action:"query"})` (schema first)           |

### Filtering, run-state, and lean reads

The list and insights routes carry rich filters — narrow at the source instead of pulling everything and sifting. The exact semantics are in each route's description (`describe` shows them); the planning-level facts:

- **By name:** `name_contains` / `name_not_contains` (case-insensitive substring, arrays OR'd; must match a `contains` term AND avoid every `not_contains` term).
- **By delivery (insights):** `had_delivery: true` returns only rows that spent/served in the window regardless of current on/off state — this is how you find ads that **spent in the last 30 days but are paused now** (insights are keyed to the window, not current status). `min_spend: <dollars>` sets a spend floor.
- **Run-state — read `delivery_status`, never raw `status`.** Meta leaves a finished entity at `status: ACTIVE` (it has no "completed" field), so a campaign that ran its course reads `status: ACTIVE` but `delivery_status: COMPLETED`. Values: `ACTIVE`, `PAUSED`, `COMPLETED`, `SCHEDULED`, `IN_PROCESS`, `WITH_ISSUES`, `PENDING_REVIEW`, `DISAPPROVED`, `NOT_DELIVERING` (billing issue), `ARCHIVED`, `DELETED`. The `status` filter still matches Meta's configured status, so `status: ACTIVE` lists include completed entities — use `delivery_status` to separate live from finished. Ads inherit a finished/paused parent. Google campaigns carry `delivery_status` too (Google's `primary_status`: `ELIGIBLE`, `PAUSED`, `ENDED`, `NOT_ELIGIBLE`, `LIMITED`, `MISCONFIGURED`) — a Google campaign at `status: ENABLED` with a past end date reads `delivery_status: ENDED`, not active.
- **Pull only the fields you need (`params.fields` allowlist):** returns only those fields (plus `id`). Ad-set `targeting` is **opt-in** (can carry thousands of ZIPs) — request it explicitly with `fields: ["targeting"]`. On ads, omit `creative` to skip the large creative blob.
- **Response size (insights):** rows return deduped top-level metrics and by default omit the heavy raw `actions[]`/`action_values[]` arrays. Pass `include_actions: true` only for a non-standard event. Broad ad-level name queries auto-use a lean field set (dropping Meta's ranking diagnostics) to dodge Meta's data-volume limit; request rankings explicitly via `fields` if needed.

### Product-Level Analysis (catalog ads)

Pass `params.breakdowns: "product_id"` to any Meta insights lens to split rows per product. Each row's `product_id` value is `"<retailer_id>, <product name>"` (merchant SKU + name), so you can often rank from the breakdown alone. To enrich winners with price/URL/image/stock, pass the raw breakdown values to `query({channel:"meta", resource:"catalog_product", mode:"list"})` via `params.retailer_ids`. Scope the breakdown to a `campaign_id`/ad-set and add `sort: "spend_descending"` — at account level a big catalog returns thousands of rows; use an insights job for a full-catalog sweep.

### Copy & Headline Analysis

`insights({channel:"meta", lens:"performance_by_copy"})` rolls primary-text and headline performance up across an ad set/campaign — pull it **once** instead of every ad. Scope with `params.adset_id` (or `campaign_id` / `ad_ids`). Defaults are lean: `view:"aggregated"` with the top 15 variants per ranking — pass `view:"ads"`/`"both"` or a higher `variant_limit` only when you need per-ad rows, and scope those to a campaign/ad set. Every row carries `optimization_result` — the count of the ad set's optimized event, named by `optimization_result_metric`. **Judge copy on that metric, not just CTR/ROAS.** Param detail (`view`, `variant_limit`) is in the route description.

## META — Create

The hierarchy: **Campaign → Ad Set → Ad → Creative**. Build top-down. All creates are `create({channel:"meta", resource:…, mode:"single"/"bulk", params:{…}})`.

### Campaign

`resource:"campaign"` — always PAUSED. Required: `name`, `objective` (`OUTCOME_SALES`, `OUTCOME_LEADS`, `OUTCOME_TRAFFIC`, `OUTCOME_ENGAGEMENT`, `OUTCOME_AWARENESS`, `OUTCOME_APP_PROMOTION`). **If you set `daily_budget`/`lifetime_budget` on the campaign (CBO), `bid_strategy` MUST live on the campaign, not the ad set.** Default `bid_strategy` to `LOWEST_COST_WITHOUT_CAP` unless the user asks for a cap. (Campaign budgets are in **dollars**.)

### Ad Set

`resource:"adset"`. Required: `name`, `billing_event`, `optimization_goal`, `targeting`. **For conversion campaigns (`OUTCOME_SALES` + `OFFSITE_CONVERSIONS`), `promoted_object` with `pixel_id` + `custom_event_type` (e.g. `PURCHASE`) is mandatory** — fetch the pixel via `query({resource:"pixel", mode:"list"})` first. Custom pixel event → `custom_event_type: "OTHER"` + `custom_event_str`; Custom Conversion → `custom_conversion_id`. When restructuring an existing ad set, read its `promoted_object` (`query({resource:"adset", mode:"get"})`) and carry it over verbatim. `is_dynamic_creative` and `destination_type` are immutable after creation (Meta returns 200 but silently drops them on updates). Attribution: conversion/value-optimized ad sets default to **7-day click + 1-day view** (the Ads Manager default) when `attribution_spec` is omitted; pass `attribution_spec` explicitly to choose different windows.

**Advantage+ audience (subcode 1870227).** If a create is rejected with subcode 1870227, it needs `targeting.targeting_automation.advantage_audience` — the error message tells you the exact retry (set `0` to honor your targeting exactly, `1` to let Meta deliver beyond it; use `targeting.targeting_relaxation_types` to expand one audience but keep another strict). Meta only requires this on create, and only when you hand-set age/gender/detailed targeting or attach a custom audience.

### Ad — Default Shape (5×5)

The recommended default is what modern Ads Manager emits: a single image-or-video creative with placement variants, 5× text rotation, and IG enabled.

```
create({channel:"meta", resource:"ad", mode:"single" or "bulk", params:{
  page_id            (from default_page_id)
  instagram_actor_id (from default_instagram_id — REQUIRED for IG placements)
  image_hash         (feed) + story_image_hash (9:16, OR omit → Meta auto-crops)
  bodies:    [up to 5 primary text variants]
  headlines: [up to 5 headline variants]
  link, call_to_action, …
}})
```

Always generate **5 body + 5 headline variants** and write **real, crafted copy — never lean on AI text generation.** The `text_generation` creative feature only expands single-text ads; on the 5×5 rotation it is inert (no single seed), so treat it as a no-op even when an account has it enabled. This 5×5 pattern is **PAC** (Placement Asset Customization), not DCO — it builds an `asset_feed_spec` internally but needs no Dynamic Creative ad set and is unaffected by the DCO-creation deprecation. Meta surfaces per-variant performance via the `performance_by_copy` lens.

**Catalog ads (DPA / Advantage+ Catalog):** same route — presence of `product_set_id` in params selects the catalog path (single `{{product.name}}`-style message; one-element `primary_texts`/`headlines`).

### Multi-Media vs Flexible (one ad, up to 10 assets)

Both put up to 10 images/videos in one ad — pick by what you want Meta to do:

- **Multi-media** — Meta serves the single best-fit asset per person/placement (stays a single-media ad). **Any objective.** Reach for "one ad with these N images/videos, show the best one." Set `format: "multi_media"` + `multi_media_images` / `multi_media_videos`.
- **Flexible** — Meta *recombines* media + text and renders as single image, video, **or carousel** per placement. It is the **Dynamic Creative replacement**, but **only on `OUTCOME_SALES` / `OUTCOME_APP_PROMOTION`** — check the campaign objective first. Set `format: "flexible"` and pass **all** media in `flexible_images` / `flexible_videos` (NOT a single `creative.image_hash` — that yields a 1-image ad).

Both support Advantage+ enhancements via `creative.creative_features`, so cloning either with a `detail:"full"` ad read reproduces them. Param shapes, the advanced `creative_asset_groups_spec` (flexible) / `media_sourcing_spec` (multi-media) passthroughs, and the "created synchronously, not batched — expected, not an error" note all live in the create-ad route description (`describe({tool:"create", channel:"meta", resource:"ad", mode:"single"})`).

**DCO creation is deprecated.** `format:"dynamic"` and any `asset_feed_spec` passthrough are rejected with a pointer to `format:"flexible"`. Reading/analyzing legacy DCO ads is unaffected (ad reads, the creative analyzer, and `performance_by_copy` still surface their `asset_feed_spec`).

**Format Automation (optional).** `creative.format_transformation_spec` opts one base creative into extra delivery formats (carousel, collection, single media, video slideshow) — off by default, set only when asked. Each data source needs its companion config or Meta rejects; catalog `product_extensions` auto-defaults to single-media + carousel + collection. Full matrix and the `sa_collection`-vs-`da_collection` rule are in the create-ad route description.

### Scheduling & launch timing

Three mechanisms — pick by scope:

- **Whole campaign, one-time launch:** create with a future `start_time` (reads `delivery_status: SCHEDULED` until then). **Do NOT try to reschedule an already-started campaign** with `start_time` + `status: ACTIVE` — Meta can't move a start_time once passed (it silently keeps the old one) and `status: ACTIVE` goes live immediately. The campaign-update route returns a `warning` when Meta ignores a `start_time` change — surface it instead of reporting a scheduled launch.
- **Single ad, independent window:** `ad_schedule_start_time` / `ad_schedule_end_time` (ISO 8601) on ad create/update routes, single or bulk (pass `null` on updates to clear). **Meta only honors these on `OUTCOME_SALES` / `OUTCOME_APP_PROMOTION`** — check the objective first.
- **Recurring on/off (dayparting) or launching an already-started campaign:** a scheduled ad rule (Automate). The rules engine has no calendar-date/one-time schedule — only recurring times.

### Promoting existing posts

- **Existing IG post:** `source_instagram_media_id` + `instagram_actor_id` + `page_id` (most campaigns also need `call_to_action` + `link` — match sibling ads). Source the media id from `query({channel:"meta", resource:"instagram_media", mode:"list"})`: no `media_id` lists recent posts (lean — set `include_media_urls:true` for CDN links); pass a `media_id` for full detail.
- **Existing FB post:** `object_story_id` + `page_id`.

### When To Use Which Ad Call

| Situation                                                        | Call                          |
| ---------------------------------------------------------------- | ----------------------------- |
| 1–2 ads, OR creative from a public `image_url`                   | `create` ad `mode:"single"`   |
| 3+ net-new ads with staged `image_hash` / `video_id`            | `create` ad `mode:"bulk"`     |
| One ad, best **single** asset (any objective)                    | `mode:"single"` + `format:"multi_media"` |
| One ad, Meta **mixes** media+text → single/video/carousel (SALES/APP) | `mode:"single"` + `format:"flexible"` |
| Fan an approved ad's creative across many ad sets                | `duplicate({channel:"meta", resource:"ads"})` |
| Clone an ad's exact setup but swap in new media                  | `query` ad `mode:"get"` + `detail:"full"` → `create` ad `mode:"bulk"` |
| Update many existing ads (pause, swap creative, …)              | `update` ad `mode:"bulk"`     |
| Edit text/URL on an existing ad, same creative shape             | `update` ad `mode:"single"`   |

**Cloning an ad exactly (swap only the media).** `duplicate` keeps the original media and the ad-update route is format-preserving, so for "same ad, new image" read the parent with **`detail:"full"`** first — the default ad read omits the Advantage+ enrollment, pinned catalog **product set**, and **site links**, so a summary clone silently falls back to account defaults. Workflow: full-detail read once → for each new ad, swap `image_hash` / `video_id` (and `story_image_hash`) and forward the rest of the `creative` unchanged, **passing the entire `site_links` list** (creatives carry 10–15; sending a few shrinks the clone). Verify by diffing `creative.advantage_plus_spec` (the complete read-only enrollment matrix) between parent and clone.

**Copy-update constraints on the ad-update route** (it is format-preserving — a single-text ad stays single-text, a rotation ad stays rotation; passing `primary_texts`/`headlines` on a single-text ad is rejected). Copy **cannot** be updated on three kinds — skip them when sweeping text edits across an ad set, and recreate to change their copy:
- **Existing-post / boosted ads** (`object_story_id` / `source_instagram_media_id` creatives) — text belongs to the post. `name`, `status`, schedule still work.
- **Flexible** ads (texts live in `creative_asset_groups_spec`).
- **Multi-media** ads (texts live in `media_sourcing_spec`).

**Catalog-ad copy:** single message only (template params like `{{product.name}}` allowed) — one-element `primary_texts`/`headlines`; multi-element arrays are rejected. When a full ad read shows multiple bodies, those are Ads Manager AI-generated text options the API can't set — the first update is rejected with an explanation; relay it, and only if the user consents to losing them permanently, retry with `confirm_drop_ai_text_options: true` (never pass it preemptively).

## META — Creative Pipeline

Staging into Xylo and pushing to Meta are **two separate steps**. The `media` staging actions put creatives in the Xylo dashboard (not on Meta yet); **`media({action:"finalize"})` is the only step that uploads to the live Meta ad account.** For more than one or two assets, **never loop single-image/video uploads** — use staging. All calls are `media({channel:"meta", action:…, params:{…}})`.

1. **Stage, pre-grouped.** Pick the entry point:
   - **Filesystem agent (Claude Code, Cursor):** review the images, decide which are placement variants of the same creative (feed + its 9:16 story), then `action:"upload_directory"` with a grouping plan; PUT each returned URL in parallel with `curl`.
   - **Any agent:** `action:"bulk_upload_urls"` with `[{filename, mime_type, group?}]` (≤100/call); a shared `group` label makes 2+ files one creative set. PUT in parallel.
   - **Single public URL/base64:** `action:"upload_image"`.
   - **claude.ai (no filesystem/HTTP):** have the user upload + group in the dashboard **Media tab**, then resume at the review step.
   - Assets without a `group` label auto-pair (feed + 9:16 of the same type) by filename or perceptual hash; explicit labels always win. Caps: 30 MB images, 300 MB videos; URLs expire in 1h. (Full grouping/auto-pair rules are in the route descriptions.)
2. **Review.** `query({channel:"meta", resource:"uploaded_media", mode:"list"})` lists staged assets grouped with `preview_url`s (and dimensions). Use `previews:"inline"` (keep `limit ≤ 10`) to actually see them. Adjust with `media({action:"group"})` / `media({action:"rename_group"})`.
3. **Finalize.** `media({action:"finalize"})` ONCE with `asset_ids` (or omit to push all unfinalized). Returns each asset's `meta_ref` (`image_hash` for images, `video_id` for videos). Already-finalized assets are skipped. If the call times out client-side on a very large batch, the server keeps uploading — re-run the same call and re-list `uploaded_media` until every asset has a `meta_ref`.
4. **Use in ads.** **Each GROUP becomes ONE ad:** the feed asset's `meta_ref` → `image_hash`, the 9:16 asset's → `story_image_hash` on the same spec (don't make one ad per asset). Pass `meta_ref` into bulk ad creation.

For Page/IG posts (not ads), use the `preview_url` from the staged-media list, not `meta_ref` — image hashes are ad-only.

## Generating Creative (AI)

`media({channel:"meta", action:"generate"})` creates new ad images with FAL-hosted models (`gpt-image-edit`, `nano-banana-pro`, `nano-banana-pro-edit`, `nano-banana`, `nano-banana-edit`, `gpt-image`, `flux-pro-ultra`, `flux-kontext`, `recraft-v3`). Each generation is billed in **generation credits** (per-model price, drawn from the org's monthly plan allowance and then its purchased balance); every response reports what's left. Headline routing: static product/promo ads with in-image text → `gpt-image-edit` (GPT Image 2: product + logo references AND best-in-class layout/text in one pass, every key ad ratio incl. 4:5/9:16); iterate on an output the user wants changed with an edit model referencing the previous image, never a from-scratch regenerate. Both GPT Image 2 models (`gpt-image-edit`, `gpt-image`) take a `quality` param defaulting to `"medium"` (20 credits/image) — if small text or a logo comes back imperfect, re-run the SAME call with `quality:"high"` (45 credits/image, crispest text rendering); `"low"` (10 credits) is for rough drafts. Start at medium; pay for high only when a result needs it.

1. `accounts({action:"read_brief"})`: the `brand_profile` (urls incl. the brand website, colors, fonts, voice, product truths), the saved `brand_assets` ids (logo, exact product shots), both brand-level and shared across every account attached to the same brand, and the account's `winning_ads` with their descriptions for style/reference matching.
2. `knowledge({topic:"creative_generation"})`: read this before every generate call. It returns the current credit cost, aspect ratios, and reference-image support per model, the decision order between them, and per-model prompting technique (layered art direction: subject, surface, lighting, exact in-image copy in quotes with layout positions, finish). Prompt like a creative director — a bare "product on black with SAVE 20%" yields a flat, generic card.
3. Classify the request and gather references before prompting: an exact product or logo in frame needs the matching brand asset id; "match our style" needs a style reference or a winning ad's own image; remixing a specific ad needs that ad's real media (`query({channel:"meta", resource:"ad_media", mode:"get"})` download URL, saved via `media({action:"save_brand_asset"})`); a pure lifestyle scene or typography card needs no reference, just brand colors and tone inlined in the prompt. If a needed reference is missing from `brand_assets`, acquire it first: pull it from a winning ad's media, save a public image URL from the brand's website with `media({action:"save_brand_asset"})`, or ask the user to upload it (Dashboard, Brand Profiles, Assets tab) or share a URL. Never generate an exact-product or logo request from a text description; approximations are off-brand.
4. `media({action:"generate", params:{model, prompt, brand_asset_ids, aspect_ratio, num_images}})`: inline the relevant brand facts into `prompt` (the model knows nothing about the brand on its own) — including the account's `audience_profile` (the dominant converting demographic from `read_brief`), so the people, tone, and styling match the real buyer, not a generic guess — and pass `brand_asset_ids` as reference images on edit-capable models (`gpt-image-edit`, `nano-banana-pro-edit`, `nano-banana-edit`, `flux-kontext`) so a logo or product renders exactly instead of an approximation. `brand_asset_ids` come from step 1 or from `media({action:"save_brand_asset"})`. `num_images` gives near-identical variants of ONE prompt — for distinct creative directions make separate calls with genuinely different prompts (default: 2–3 distinct concepts × 1 image, then variants of the user's pick).
5. Review the returned preview URLs with the user before finalizing (generated in-image text can still be misspelled).
6. **Generation is not publishing.** A request to generate images is complete when the user has the images — do NOT suggest building ads from them, propose campaigns/ad sets to put them in, or fold "and I'll set them up in the account" into other suggestions. The user approving an image ("yes, I like it", "accept your suggestions") approves the *image*, never an ad build. Only when the user explicitly asks to run/publish/put up the creative: `media({action:"finalize"})` to push the chosen output to the live Meta ad account, then `create({resource:"ad"})` to build the ad (lands PAUSED like every other create).

Every generated image also lands in the brand's gallery in Xylo — send the user a direct link to look at them rather than only pasting preview URLs: `https://xylomcp.com/dashboard/brands?brand=<brand_id>` (Assets → AI generated). From there the "Save" button pushes a pick to Meta permanently and the resulting `image_hash` references it going forward, the same as any other finalized asset.

**Product catalog as reference images.** For a product already in a Meta catalog, skip external references entirely: find it with `query({channel:"meta", resource:"catalog", mode:"list"})` then `query({channel:"meta", resource:"catalog_product", mode:"list", params:{q:"<product name>"}})`, then `media({channel:"meta", action:"stage_from_catalog", params:{product_id}})` — free, no generation credits — to pull its photos (main image + additional angles) into 24h staging. The response includes an inline preview of every staged image; look at each and keep only clean shots (plain/studio background, no text overlay, no lifestyle clutter), preferring distinct angles, up to about 4. Pass the kept ids as `media_asset_ids` to `media({action:"generate"})`, and write the prompt from the product's real catalog name, description, and brand rather than guessing. Staged assets expire in 24h — re-stage on demand for a later session.

**Asset refs.** The dashboard's "Copy ID for Claude" button gives the user a ref like `brand_asset:{id}`, `winning_ad:{id}`, or `generated:{id}` to paste directly into a prompt. Resolve any of these with `media({action:"get_asset", params:{ref}})` to get the underlying asset (URL, dimensions, description) before using it as a reference. Refs are globally unique, so the ref alone is enough — no platform or account_id needed.

**Full commercials (Director mode).** When the user wants a real multi-scene video ad — recurring characters, exact product fidelity, a story — read `knowledge({topic:"video_ad_production"})` FIRST and follow it: intake interview, locked reference sheets (product/character/location/layout, saved as brand assets with an @name map), a style prefix the user chooses (house default, brand-derived, or custom), a named shotlist, shot-by-shot generation with positional @Image references, then `media({action:"stitch"})` to join approved clips into the final file. This is a collaboration with the user at every gate, not a one-shot generation.

## META — Bulk Operations

Xylo's bulk routes use Meta's Batch API (one read for a filter + chunked 50-op parallel batches — ~5 HTTP calls for 200 entities). Bulk writes are `update({channel:"meta", resource:"campaign"/"adset"/"ad", mode:"bulk"})`; bulk pause/resume flips are also `status({channel:"meta", resource:…, mode:"bulk", params:{<entity>_ids, status}})`.

- **Filter mode:** `params.filter={name_contains, status, objective, …}` + `params.updates={…}`. **Always `dry_run: true` first** on a destructive filter (PAUSED, budget cuts) — verify the match set.
- **Explicit mode:** `[{id, …payload}, …]` for per-entity changes.
- **Partial failures are expected** — inspect both `succeeded[]` and `failed[]`. Timed-out writes are auto-retried once; recovered ids appear in `succeeded_ids` + `retried_ids`. Anything left in `failed[]` definitively didn't apply; re-running just those ids is safe (updates are idempotent). Limits: 20 default, 200 max.
- **Bulk ad creation and `duplicate` are self-healing and idempotent** (by ad name / by copy). Report the `summary` (`requested` / `created` / `skipped_existing` / `failed`) — it is the true count. Ops that time out inside Meta's batch are auto-verified against the target ad set and flagged `verified_after_timeout` / `retried` — those ads are REAL, never recreate them. If the tool call itself times out client-side, call it again with the **exact same specs** — existing ads are skipped, only missing ones created. **Never rename specs when retrying** (renaming defeats the duplicate protection).
- **Duplicated legacy creatives may come back `rebuilt: true`** — Meta refused its native copy, so Xylo reconstructed the full creative (media pool, Advantage+ enrollment, site links, pinned product set, tracking specs). If anything could not be carried over, the entry also carries `fidelity_warnings` naming exactly what — surface those to the user verbatim. Partnership / branded-content ads cannot be duplicated without partnership authorization; Xylo rejects them up front with a clear error instead of a half-built copy.

## META — Automate

Xylo exposes Meta's full Ad Rules Engine (= Ads Manager "Automated Rules") and Value Rules (bid multipliers). Reads live on `query({channel:"meta", resource:"ad_rule"/"value_rule_set"})`; writes on `automation({channel:"meta", resource:"ad_rule"/"value_rule_set"})`. This is where Xylo turns from "do it now" into "keep doing it" — pause losers, scale winners, daypart, alert, rebalance, bid more for high-value audiences.

### Ad rules — the mental model

Every rule is three parts: **Evaluation** (`applies_to` = a `level` of AD/ADSET/CAMPAIGN, optionally scoped to `ids`; plus `conditions` + a `time_window`) → **Action** (`rule_action`) → **Schedule** (`schedule`). Read actions (`query`): `list`, `get`, `history`, `active`, `count_by_type`. Write actions (`automation`): `create`, `update`, `enable`/`disable`, `delete`.

Define a rule two ways: **(A) ergonomic** — `applies_to` (required; omit `ids` to apply to ALL objects at that level, including future ones), `conditions` (`[{metric, operator, value}]`, AND-combined), `time_window` (required when conditions present), `rule_action`, `schedule`. **(B) raw escape hatch** — pass `evaluation_spec`/`execution_spec`/`schedule_spec` verbatim (overrides ergonomic); needed for trigger (real-time) rules, `ROTATE`, and advanced multi-level/aggregate/formula filters. The full grammar (metrics, operators, `change`/`rebalance`/`schedule` shapes, `time_window` presets) is in the route description.

**Safety (do not skip):**
- **New rules are created DISABLED.** After `create`, tell the user in plain language exactly what it watches and does, then `enable` (or pass `status: "ENABLED"` on create only when they explicitly ask). `enable`/`disable` self-verify — a rare `verified: false` + `warning` just means Meta is lagging; call the same action again.
- **Money is in CENTS** (both `conditions` thresholds and `change.amount` with `unit: "ACCOUNT_CURRENCY"`): `5000` = $50.00. Responses annotate an `≈ $X.XX` value — **always say the dollar amount back to the user.** (Rules use raw cents; campaign budgets are dollars.)
- **A PAUSE rule cannot filter on a cost metric** (`cpa`, `cost_per_*`, `cpc`, `cpm`) — Meta error 2703; the route pre-rejects it with the fix. Gate on a volume+outcome combo instead (spend high AND results = 0).
- **Start narrow** (explicit `ids` for a first version; widen to a whole `level` only once trusted). No dry-run / "run now" exists — verify after the fact with the `history` read action.
- **Maintenance:** a `HAS_ISSUES` rule → `get` it, read `disable_error_code`, fix, `update` (update **replaces the whole spec** — re-supply `applies_to` when you change `conditions`).

**Recipes** (each is `automation({resource:"ad_rule", params:{action:"create", …}})`; all land DISABLED — `enable` once the user approves). These four cover the distinct shapes; other actions (`NOTIFY`, `CHANGE_BID` toward a target, `REBALANCE`, frequency-based `PAUSE`) follow the same structure — swap the metric/action and use the `change`/`rebalance` schemas from the route description.

```
1 · Auto-pause ads with no conversions after real spend (can't gate on CPA — error 2703):
  applies_to: {level:"AD"}
  conditions: [{metric:"spent", operator:"GREATER_THAN", value:3000},   // $30 (cents)
               {metric:"results", operator:"EQUAL", value:0}]
  time_window:"LAST_3D"  rule_action:{type:"PAUSE"}  schedule:"DAILY"

2 · Scale ad-set budget when ROAS is strong (+20%/day, $100 cap, ≤ once/day):
  applies_to: {level:"ADSET"}
  conditions: [{metric:"website_purchase_roas", operator:"GREATER_THAN", value:2}]  // ROAS = decimal multiple
  time_window:"LAST_7D"
  rule_action:{type:"CHANGE_BUDGET", change:{amount:20, unit:"PERCENTAGE", limit:10000}, action_frequency:1440}
  schedule:"DAILY"
  // CBO campaign? Budget lives on the campaign — use type:"CHANGE_CAMPAIGN_BUDGET" + applies_to:{level:"CAMPAIGN"}.

3 · Dayparting (pause an ad set weekday evenings; no conditions for a pure schedule pause):
  applies_to: {level:"ADSET", ids:["<adset_id>"]}
  rule_action:{type:"PAUSE"}
  schedule:[{start_minute:1080, days:[1,2,3,4,5]}]   // 18:00 Mon–Fri; pair with an UNPAUSE rule to bring them back

4 · Real-time auto-pause (trigger rule — raw, no schedule) when 3-day cost/purchase crosses $15:
  evaluation_spec:{evaluation_type:"TRIGGER",
    trigger:{type:"STATS_CHANGE", field:"cost_per_purchase_fb", operator:"GREATER_THAN", value:1500},
    filters:[{field:"entity_type", operator:"EQUAL", value:"AD"},
             {field:"time_preset", operator:"EQUAL", value:"LAST_3_DAYS"}]}
  execution_spec:{execution_type:"PAUSE"}
```

An `UNPAUSE` rule always shows a "paused/eligible objects" condition in Ads Manager (Meta auto-adds it) — expected, not mis-scoping.

### Value Rules

Bid **multipliers** telling Meta which audience slices are worth more, applied at the ad-set level (they don't *exclude* anyone, they bid relatively more/less). Use for "pay more for high-value audiences." A rule set holds up to 10 rules; each rule up to 4 AND-combined criteria; 6 rule sets per account; first matching rule wins (array order = priority). **Eligibility:** the ad set must use auto-bid (`LOWEST_COST_WITHOUT_CAP`) or `COST_CAP`. Reads (`query`): `list`, `get`. Writes (`automation`): `create`, `update`, `delete`, `attach` (`adset_id` + `value_rule_set_id`), `detach`. Editing is read-modify-write (keep each kept rule's `id`, omit `id` to add, drop from the array to remove).

```
Recipe — bid +30% for 25–44 men in the US (automation, resource:"value_rule_set"):
  action:"create"  name:"High-LTV core"
  rules:[{name:"M 25-44 US", adjust_sign:"INCREASE", adjust_value:30,
          criterias:[{criteria_type:"AGE", criteria_values:["25-34","35-44"]},
                     {criteria_type:"GENDER", criteria_values:["MALE"]},
                     {criteria_type:"LOCATION", criteria_values:["US"], criteria_value_types:["LOCATION_COUNTRY"]}]}]
Then: {action:"attach", adset_id:"<id>", value_rule_set_id:"<id>"}
```

`adjust_value` is a percent (INCREASE 1–1000, DECREASE 1–90); the criteria dimensions and operators are in the route description.

## Creative Intelligence

Three surfaces: the **analyzer** (what a creative IS and why it performs), the **diagnostic** (why it's failing/winning, delivery + creative), and **frameworks** (how to make it better). All read-only; none touch the ad account.

### Analyzer — `creative({action:"analyze"})`

Turns any creative into ~30 structured Gemini-powered fields. Use it to explain **WHY** a creative works, not whether (that's the insights lenses' job).

| Input                                           | Call                          |
| ----------------------------------------------- | ----------------------------- |
| Meta `ad_id` or `creative_id`                   | `creative({action:"analyze", channel:"meta"})`    |
| Google asset `resource_name` (image or YouTube) | `creative({action:"analyze", channel:"google"})`  |
| TikTok `ad_id` or `video_id`                    | `creative({action:"analyze", channel:"tiktok"})`  |
| Any public image/video URL                      | `creative({action:"analyze_url"})`   |

Default to `full` mode (Gemini Pro, ~20–30s on video, reasoning fields); switch to `fast` for broad sweeps. Same asset + prompt version + mode = same cache key; `refresh: true` re-runs. The highest-signal fields are the **psychology block** (each with a `_reason` in full mode): `awareness_level` (Schwartz's 5: unaware / problem_aware / solution_aware / product_aware / most_aware), `life_force_8_triggered` (Whitman's LF8), `mindstate_targeted` (Leach's 9), `regulatory_focus` (promotion / prevention / mixed), `dominant_emotion_targeted` + `emotion_trigger_alignment`. Also: content, hook breakdown, production fingerprint, timing markers (video), conversion architecture. The response is self-describing (field list + cache fields come back on every call).

**Carousels:** one card per call — pass `card_index` (default 0). **Bulk: apply Pareto first** — sort by spend, keep the top ~90% of cumulative spend; analyzing $5–$20 lifetime ads wastes inference below the confidence floor (~5,000 impressions / $30 spend). The can't-see list (DPA/catalog, TikTok single-image, comment sentiment, fatigue → use the diagnostic, A/B comparison → diff yourself) and the degraded-thumbnail fallback are documented at call time in the response — see Troubleshooting for the degraded-mode remediation.

### Diagnostic — `creative({action:"diagnose", channel:"meta"})`

Answers "why is this ad failing or winning?" — combines the analyzer's structured fields with delivery signals (hook_rate, hold_rate, frequency, CPM, CTR, ROAS) and routes the ad into one of five buckets, each with a concrete fix citing specific analyzer fields. Auto-bootstraps the analyzer on cache miss (you don't need to call the analyzer first). Inputs: `ad_id` + `ad_account_id` (required), optional `date_preset` or `date_from`+`date_to` (default `last_14d`). Buckets (first-match wins): `wrong_audience_or_fatigue` (frequency/CPM — fatigue precedes creative quality), `poor_hook`, `good_hook_poor_hold`, `good_creative_weak_cta`, `average_everything`. Confidence `high` needs ≥5,000 impressions AND ≥$30 spend — always check `caveats[]`. **Pareto first**, same as the analyzer. The metric definitions and bucket thresholds come back in the response.

**Canonical workflow — "why is this campaign struggling?":** `insights lens:"top_performers"` (or `performance_by_copy`) to find the spenders → diagnose each → cluster by bucket (many `wrong_audience_or_fatigue` = targeting problem, not creative; many `good_hook_poor_hold` = campaign-wide pacing pattern) → act per the bucket's fixes.

### Frameworks — `knowledge({topic:"creative_frameworks"})`

The analyzer tells you what a creative IS; the frameworks brain tells you how to make it BETTER. Call it whenever you move from analysis to creation — variations, hook rewrites, UGC scripts, fixing a salesy/low-retention cut, net-new concepts. Default call (no `params.topics`) returns the complete expert brain (hook formula, diagnostic funnel, viral scripting, copywriting, Metadeception native-feel system, Zeigarnik open-loop retention, 100 hook templates, checklists, truthfulness guardrail) — the recommended usage. Scope with `topics` only for bulk/cost (the diagnostic's fix bucket maps to a topic: `poor_hook → hooks`, `good_hook_poor_hold → open_loops + scripting`, `good_creative_weak_cta → copywriting`). **Always craft the real thing** — original scripts and concepts, never auto-generated filler.

**Canonical "explain why our top creative works":** `top_performers` by the optimized metric → analyze each winner → diff the structure (shared `awareness_level`? recurring `visual_hook`? common LF8? same `pov`/`regulatory_focus`?) → frame the recurring pattern as a testable hypothesis, then pull the frameworks brain and write net-new concepts holding the winning DNA constant while varying one element at a time.

## Google Ads

Hierarchy: **Campaign → Ad Group → Ad/Keyword**. Campaigns default to PAUSED. Everything is `channel:"google"`; account resolution returns a `customer_id`.

- **Conversion lag (read before judging recent performance):** Google attributes each conversion to the **click date**, so conversion metrics (`conversions`, `conversions_value`, `all_conversions`, `cost_per_conversion`, ROAS) for the **last ~7 days are incomplete** and keep rising for days/weeks. `cost`/`clicks`/`impressions` are unaffected. When the window touches that zone, Google insights responses carry a ⚠️ `conversion_attribution` warning with `reliable_through`. **Never pause or scale on conversion or CPA/ROAS data from the last week** — anchor conversion decisions to a window ending ≥7 days ago; use recent days only for cost/click/impression trends.
- **Bidding:** new advertisers start with `MAXIMIZE_CLICKS` or `MAXIMIZE_CONVERSIONS`; switch to `TARGET_CPA`/`TARGET_ROAS` only after 30+ conversions.
- **Ad copy:** RSAs need 3+ headlines and 2+ descriptions (`create({channel:"google", resource:"ad"})`, `RESPONSIVE_SEARCH_AD`). Ad content is immutable post-create — build a new ad and remove the old to change copy. Pausing/resuming an ad is `status({channel:"google", resource:"ad"})`.
- **Performance Max:** create campaign → at least one `asset_group` → link assets with `update({channel:"google", resource:"asset_group_asset", action:"link"})`. Strict asset minimums — check returned errors.
- **Keyword research:** `query({channel:"google", resource:"keyword_ideas"})` — discover new keywords + search volumes (Keyword Planner) from seed `keywords` and/or a landing-page `page_url`. Returns avg monthly searches, competition, and top-of-page bid range. Defaults to US + English. Scope volumes to a place with `geo_names:["Austin, TX","California"]` (resolved for you) or numeric `geo_target_ids`; volumes are aggregated across the geos you pass, so call once per place for a per-city breakdown. Keyword Planner reports down to **city/metro** level — not individual ZIP/postal codes. Use it to scope a Search build or expand an ad group before creating keywords.
- **Geo resolver:** `query({channel:"google", resource:"geo_target", mode:"list", params:{names:["Austin, TX"]}})` — resolve place names to geo target constant IDs (with canonical_name, target_type, reach) for `geo_target_ids` on keyword research or campaign geo targeting.
- **Negative keywords:** prefer shared lists (`create({resource:"negative_keyword_list"})` + `update({resource:"negative_keyword_list", action:"link_to_campaign"})`) over per-campaign negatives for broad exclusions.
- **Recommendations:** `query({resource:"recommendation", mode:"list"})` → present → `update({resource:"recommendation", action:"apply"})` only after explicit approval (one-way).
- **Quality:** `insights({channel:"google", lens:"quality_scores"})` before suggesting bid changes on low-performing keywords.
- **Experiments:** `create({resource:"experiment"})` in SETUP; `update({resource:"experiment", action:"promote"})` is one-way and replaces the original campaign.
- Conversion priority (primary vs secondary) can't be set via API — direct the user to the Google Ads UI.

## TikTok Ads

Hierarchy: **Campaign → Ad Group → Ad**. Campaigns default to DISABLE. Everything is `channel:"tiktok"`; resolve an `advertiser_id` first. TikTok status values are `ENABLE`/`DISABLE` (`status({channel:"tiktok", …})`).

- Objectives: `REACH`, `TRAFFIC`, `VIDEO_VIEWS`, `LEAD_GENERATION`, `CONVERSIONS`, `APP_INSTALL`, `PRODUCT_SALES`. Ad-group optimization goals: `CLICK`, `IMPRESSION`, `REACH`, `CONVERSION`, `INSTALL`, `VIDEO_VIEW`, `LEAD_GENERATION`.
- **Standard performance** (`insights({channel:"tiktok", lens:"campaign"/"ad_group"/"ad"})`) is for regular auction campaigns only — **not** GMV Max (below).
- **Smart+** campaigns have dedicated resources — `create`/`update` with `resource:"smart_plus_campaign"` / `"smart_plus_ad_group"` / `"smart_plus_ad"`; material performance via `insights({lens:"smart_plus_material"})`.
- **Spark Ads** from creator videos: `query({resource:"spark_post", mode:"list"})` + the TCM resources (`query({resource:"tcm_creator"/"tcm_order"})`, `publish({resource:"tcm", action:"invite"/"review"})`).
- **Download ad creatives:** `query({channel:"tiktok", resource:"ad_media", mode:"get"})` (`ad_id` → temporary signed URLs — download immediately, never store; ~6h library video / ~1h Spark). Spark posts owned by a TikTok-Shop (`TTS_TT`) identity can't resolve here — pass the post id/URL instead (same route) for a stable re-hosted URL.

### GMV Max (automated TikTok Shop campaigns)

Every GMV Max campaign bids toward an ROI target (`roas_bid`) with a daily budget. **Performance comes ONLY from `insights({channel:"tiktok", lens:"gmv_max"})`** — the standard campaign lens does NOT work for GMV Max; don't call it for them. The route's description defines the exact dimension/filter/metric combination rules (which filters are required at product vs creative level, why you must request `product_name`, `cost`-not-`spend`, date caps) — follow them exactly.

Route map: list/read `query({resource:"gmv_max_campaign", mode:"list"/"get"})`; shops `query({resource:"gmv_max_store", mode:"list"})` (the entry point for a `store_id` + `store_authorized_bc_id`); targets `plan({channel:"tiktok", action:"bid_recommendation"})`; create/update `create`/`update` `resource:"gmv_max_campaign"`; status (incl. its delete op) `status({resource:"gmv_max_campaign"})`; roster `query({resource:"gmv_max_post"/"gmv_max_creator", mode:"list"})`; media `query({resource:"ad_media", mode:"get"})`. Only the creator roster returns real `@handles` (and a derived `profile_url`) — the report/posts routes expose display names, which are NOT handles. Media `download_url`s are signed and expire — expect many `download_url: null` (roster access); the reliable fallback for any public post is the ad_media route with a post id or URL (stable re-hosted URL).

**Recipe — "which creators drove revenue last month?"** (there's no standalone creator dimension): run a **product-level** report (group by `item_group_id`, filter `campaign_ids`, request `product_name`) to find top SPUs → for each campaign×SPU run a **creative-level** report (group by `item_id`, filter both `campaign_ids` AND `item_group_ids`, request `tt_account_name` + `gross_revenue`/`orders`) → aggregate the `item_id` rows by `tt_account_name`. Attribute fields (`tt_account_name`, `title`) only return with a single `campaign_ids` AND single `item_group_ids` value. Cross-reference the creator roster for clickable `profile_url`s.

## X Ads

Hierarchy: **Campaign → Ad group → Promoted tweet**. The objective lives on the ad group, not the campaign. The X API exposes an ad group through `resource:"line_item"`; use that exact technical key in tool calls. Every call goes through `x_ads({resource, action, params})`; resolve the account first with `x_ads({resource:"account", action:"list"})`, then pass its base36 id as `params.account`. Call `describe` before unfamiliar writes.

- **Campaign and ad-group management:** list, inspect, create, update, delete, and batch campaigns and ad groups. New entities land paused. Budgets and bids are expressed in dollars; raises require confirmation.
- **Promoted tweets and creative:** create a nullcast tweet, attach it to an ad group as a promoted tweet using the required `line_item_id` field, manage cards, upload media, and reuse account media. X creative analysis and the winning-ads library are not supported yet.
- **Targeting:** search locations, interests, events, devices, languages, conversations, and other targeting options; get suggestions; add or remove criteria; estimate audience size before launch.
- **Audiences:** create and manage custom audiences, upload CRM members, share audiences, and maintain do-not-reach suppression lists. Send emails and phone numbers as plaintext; Xylo normalizes and hashes them server-side.
- **Performance:** use `x_ads({resource:"insights", action:"active_entities"})` before stats, `action:"stats"` for a synchronous window up to seven days, or `action:"job_create"` plus `action:"job_status"` for longer and segmented reports. X marks recent spend as estimated.
- **Measurement and optimization:** manage web conversion tags, send conversion events, inspect change history and recommendations, work with catalogs, and run A/B tests.

## AI & Audit Tools

The named AI tools plus the cross-platform lens. Each is platform-scoped despite the name — check the tag:

- `insights({channel:"cross", lens:"cross_platform"})` *(Meta+Google+TikTok+X)* — normalized side-by-side view with per-platform and grand totals. **Google conversion lag applies:** when the window touches the last ~7 days, Google conversions are incomplete and the response prepends a ⚠️ note — caveat the Google column or shift the window to end ≥7 days ago (never compare Google ROAS raw against Meta on a recent window).
- `morpheus_audit` *(Meta or Google)* — comprehensive structural audit (Morpheus Media RMP): branded separation, prospecting consolidation, retargeting ADV+ settings, creative fatigue, audience overlap, tracking health. Pass `ad_account_id` (Meta) or `google_customer_id` (Google). Deepest review.
- `audit_campaign` *(Meta)* — quick benchmark-based score (0–100) + findings with severity.
- `optimize_budget` *(Meta)* — reallocation recs across campaigns. If campaigns use Advantage+ CBO, Meta already optimizes — recommend incremental changes (10–20%), never wholesale shifts.
- `generate_report` *(Meta)* — exec summary + key metrics + top/bottom performers + hypothesis-framed recs (default 30-day).

Call `knowledge({topic:"meta_ads"})` before any audit tool (they assume you already have that context). For per-creative structural analysis see Creative Intelligence.

## Reporting Dashboards

Build a live, block-based reporting dashboard the user can view in Xylo or hand to a client as a read-only link — a step up from a one-off `insights` pull when the user wants something they (or a client) will come back to. **Brand and Agency plans only** (and founder); Free accounts don't have this feature — if a Free-plan user asks, say so plainly rather than attempting the calls. All routes are `channel:"cross", resource:"dashboard"` (the dashboard is inherently cross-platform, matching `insights({channel:"cross"})`).

- **Create:** `create({channel:"cross", resource:"dashboard", params:{title, accounts:[{platform, account_id, name?}], spec?}})`. `accounts` must be active connected accounts of the org (from `accounts({action:"list"})`); omit `spec` to create an empty dashboard and fill it in with `update`.
- **The spec is the single artifact.** `spec = { version:1, title, date_range:{preset:"last_30d"|"last_7d"|"last_14d"|"this_month"|"last_month"|"custom"}, widgets:[{ id, type:"kpi"|"timeseries"|"bar"|"donut"|"table"|"text", title, layout:{x,y,w,h} (12-col grid), query:{ platforms, level:"account"|"campaign"|"adset"|"ad", metrics:[...], breakdown?:"time", filters?, sort?, limit? } }] }`. `text` widgets omit `query` and carry a markdown `body` instead; every other type requires `query`. A widget's `metrics` array mixes core metrics (`"spend"`, `"impressions"`, `"clicks"`, `"ctr"`, `"cpc"`, `"conversions"`, `"cost_per_conversion"`, `"roas"`) with conversion-metric objects (`{type:"conversion", platform:"meta"|"google", action, stat:"count"|"value"|"cost_per"|"value_per_cost"}`).
- **Catalog-first: never guess a conversion action.** Before binding a conversion metric, call `query({channel:"cross", resource:"dashboard", mode:"metric_catalog", params:{platform, account_id}})` — it returns the account's real Meta action types/custom conversions or Google conversion actions (plus core metrics and the valid `level`/`breakdown`/`filter` dimensions) and is the only source of truth for `action` values.
- **Sanity-check before/after writing:** `query({..., mode:"data", params:{dashboard_id, widget_ids?}})` executes the spec's widget queries live so you can see what will actually render.
- **Update / delete:** `update({..., params:{dashboard_id, title?, accounts?, spec?}})` (spec/accounts each replace the existing value wholesale); `delete({..., params:{dashboard_id}})` is permanent, gated like any other destructive call.
- **Read:** `query({..., mode:"list"})` for the org's dashboards (id, title, accounts, share status — not the spec); `query({..., mode:"get", params:{dashboard_id}})` for one dashboard's full spec.
- **Share flow:** `publish({..., action:"share", params:{dashboard_id}})` returns a public, unauthenticated `https://xylomcp.com/share/<token>` URL — logged out, not indexed by search engines. Re-sharing rotates the token and invalidates the old link. `publish({..., action:"unshare", params:{dashboard_id}})` revokes it; the dashboard itself stays accessible to the org.
- Manual edits (drag/resize/swap block type/rename/recolor/add/delete widgets) in the dashboard editor and your `update` calls write the exact same spec — there's no separate "agent version" to reconcile.

## Audiences

| Goal                              | Meta                                  | Google                          | TikTok                                  |
| --------------------------------- | ------------------------------------- | ------------------------------- | --------------------------------------- |
| Upload a CRM customer list        | `update({resource:"audience", action:"users"})` (op="add", plaintext, server SHA-256s) | `update({resource:"user_list", action:"upload_members"})` (pre-hashed SHA-256) | `update({resource:"audience", action:"append"})` |
| Build a lookalike                 | `create({resource:"lookalike_audience"})` (0.01–0.20 ratio) | (Customer Match)                | `create({resource:"audience", params:{type:"lookalike"}})` |
| Discover interests / behaviors    | `plan({channel:"meta", action:"search_targeting"})` (`type=interests`/`behaviors`/`demographics`/… — one route, all 7 types); `action:"search_geo"` for places; `action:"interest_suggestions"` to expand a known interest | (audience manager) | `plan({channel:"tiktok", action:"search_targeting"})` |
| Verify audience size before launch | `plan({channel:"meta", action:"reach_estimate"})` | (audience manager)              | `plan({channel:"tiktok", action:"audience_size"})` |

## Pixel & Conversions API

- **Meta Pixel:** one per account. `query({resource:"pixel", mode:"list"/"get"})` (base snippet on get) → `update({resource:"pixel"})` (advanced matching).
- **Server-side events (CAPI):** `tracking({channel:"meta", action:"send_events"})`. Send user data (email, phone, name) as **plaintext** — the server SHA-256s before forwarding. Do NOT pre-hash (`client_ip_address` and `fbc` go unhashed). Max 1,000 events/call.
- **Google offline conversions:** `tracking({channel:"google", action:"upload_offline"})` requires `gclid` per row.
- **TikTok events:** `tracking({channel:"tiktok", action:"send_events"})`.
- **X events:** `x_ads({resource:"tracking", action:"send_events"})`; inspect web conversion tags through the same dispatcher.

## Organic Social

Beyond ads, Xylo publishes and manages organic content — all through `publish({channel:"meta", …})` for writes and `query` for reads: create/schedule Facebook Page posts (`publish resource:"page_post" action:"create"`; scheduled list via `query resource:"page_post" mode:"list"` with `scheduled:true`), publish Instagram posts/reels/stories (`publish resource:"instagram_media" action:"publish"`; check the rate window via `query resource:"instagram_publishing_limit"`), read latest posts (`query resource:"page_post"/"instagram_media" mode:"get"`), and manage comments (`query resource:"comment" mode:"list"` to read; `publish resource:"comment" action:"moderate"` to reply/hide/delete; `action:"settings"` to toggle commenting). Use the staged asset's `preview_url` (not `meta_ref`) for post media.

## Research

### Instagram Research

Two read-only, non-account routes on `query({channel:"meta", …})`. Both need an `ig_account_id` from one of YOUR connected IG Business accounts as caller identity (from `query({resource:"instagram_account", mode:"list"})` or `default_instagram_id`).

- **Business discovery** — `query({resource:"instagram_account", mode:"get", params:{username:"<any public handle>"}})` → followers, name, bio, website, media_count; optional `include_media:true` (`media_limit ≤ 25`) for recent posts with public engagement. "How many followers does {brand} have / show {competitor}'s last 10 posts."
- **Tagged/UGC monitor** — `query({resource:"instagram_media", mode:"list", params:{tagged:true}})` — posts where YOUR connected IG was tagged by others (paginated). "UGC for our brand / who tagged us / influencer mentions."

**Meta's hard limits:** personal (non-Business/Creator) accounts aren't discoverable (`VALIDATION_ERROR`); reach/impressions/engagement-rate/demographics are NOT available for accounts you don't own — only public `like_count`/`comments_count`. First-page responses cache 1h/30min; `refresh:true` bypasses.

### Competitor Intelligence

Spy on any brand's public ads across Meta, Google, and TikTok libraries — **no connected account needed** (public data, cached 24h). All on `competitors({channel, action, params})`:

- `{channel:"meta", action:"brand_ads"}` — a competitor's live FB/IG ads by domain (`nike.com`) or page URL: copy, headlines, CTAs, landing pages, media, run dates.
- `{channel:"meta", action:"search"}` — keyword search across many advertisers ("webinar", "black friday").
- `{channel:"google", action:"brand_ads"}` — a competitor's Google ads (Search/Display/video) from the Transparency Center, by domain.
- `{channel:"tiktok", action:"search"}` → feed a result `id` into `{channel:"tiktok", action:"details"}` (full creative + targeting breakdown for one ad).

**Showing the ads to the user:** every result carries direct Meta/TikTok CDN links (`scontent…fbcdn.net`, etc.). To display them, **download the image/video files and present them as inline attachments** — do NOT embed the CDN URLs in an artifact or HTML preview, because that sandbox blocks external image hosts and every tile renders blank.

## Marketing Playbooks

Beyond ad operations, Xylo ships 20 operator-grade marketing playbooks (SOPs) served on demand — strategy and process the account brief alone can't give you. **Never load more than one or two per task.** Get the index with `knowledge({topic:"playbooks"})`; fetch one with `knowledge({topic:"playbooks", params:{playbook:"<key>"}})`.

Reach for a playbook when the user asks for strategy, process, or "how should we approach X" — not for plain execution:

- **ads** — platform selection, campaign structure, retargeting, landing-page alignment, reporting SOPs
- **ad_creative** — angle matrices + the performance-data creative iteration loop (pairs with creative analysis + generation)
- **ab_testing** — hypothesis design, sample sizing, analysis discipline
- **analytics** — tracking plans, UTM strategy, pixel/CAPI + Google conversion setup
- **competitor_research** — full competitor dossier, ad-libraries-first
- **customer_research** — JTBD, watering-hole research, personas
- **marketing_loops** — recurring scheduled-agent workflows with exact route sequences
- **cro** / **copywriting** / **copy_editing** — landing pages and the words on them
- **offers** — value equation, guarantees, bonus stacking (fix the offer before blaming the ads)
- **marketing_psychology** — behavioral-science levers for angles, offers, pricing
- **emails** / **sms** / **social** — lifecycle and organic channels
- **launch** — product/feature launch phases and checklist
- **pricing** / **content_strategy** / **ai_seo** / **marketing_ideas** — strategy beyond the ad account

Rules: run `accounts({action:"read_brief"})` before any playbook; where a playbook conflicts with `knowledge({topic:"meta_ads"})` doctrine, meta_ads wins; ad-script/hook craft stays in `knowledge({topic:"creative_frameworks"})`.

## Output Standards

- Lead with what changed and the IDs (campaign_id, adset_id, ad_id) so the user can audit.
- Present metric tables sorted descending by spend unless asked otherwise.
- When comparing platforms, normalize to the same date window and call out attribution differences explicitly.
- Frame recommendations as testable hypotheses with the expected metric and window, not directives. ("Test pausing X; expect Y to move by Z over 14 days.")
- Surface partial failures verbatim — never hide a `failed[]` array.

## Troubleshooting

The routes teach most failures at call time; this is the index. **When a call errors unexpectedly, or the answer is wrong/missing, call `send_feedback`** (include the error, calls tried, user goal, and IDs — don't ask permission, just file it).

- **Validation error with a schema + example in it** → the route told you exactly what to fix; correct the call once. If the discriminator combo was wrong, the error lists the valid values; `describe({tool:…})` shows the full route list.
- **`confirm_token` expired/mismatch** → re-run the same call without the token for a fresh preview, then confirm.
- **Rate limit / `429` / `META_RATE_LIMIT`** → Meta is throttling this token. Back off and retry after a short wait, or narrow the request. A 429 means *unknown*, not *zero* — never read it as "no pages/catalogs/data."
- **"Account not found" / wrong account** → re-run `accounts action:"list"` with a more specific `query`; if multiple match, ask the user.
- **Meta returns 200 but nothing changed** → immutable field (`is_dynamic_creative`, `destination_type`) or a `transient` partial failure from a bulk batch.
- **Ads only run on Facebook, not IG** → missing `instagram_actor_id` on the creative.
- **Ad-copy update rejected a `primary_texts`/`headlines` array** → the ad is single-text, an existing-post/boosted ad, or a Flexible/Multi-media ad — copy can't be changed in place on any of these; recreate to change copy, and skip them when sweeping text edits (see Copy-update constraints).
- **Heavy insights timing out / "please reduce the amount of data"** → `insights lens:"job" mode:"job_create"` → `mode:"job_check"` (needs the data-warehousing add-on).
- **Warehouse call returned `{ available: false }`** → the account isn't sync-enabled (or the org lacks the add-on). Use live tools; don't retry.
- **Analyzer `UNSUPPORTED_ASSET_TYPE`** → DPA/catalog ad (feed-driven), TikTok single-image (video only in v1), or no resolvable media. Don't retry; pivot to insights or pick another ad.
- **Analyzer `analysis_quality: "degraded_thumbnail"`** → Meta wouldn't return video bytes; the analysis is valid for copy/composition/awareness/psychology but caveat hook-timing and frame-by-frame claims. Follow `degraded_reason.message` / `.kind`: `meta_video_partnership_locked` = Partnership/Branded Content where the FB video node AND the automatic Instagram-side fallback both came up empty for THIS asset — per-asset, not a blanket limitation (most partnership videos analyze at full fidelity via the IG fallback); don't suggest page-admin grants, and don't tell the user partnership ads can't be analyzed — a retry or a sibling partnership ad often succeeds. `meta_video_page_admin_required` = remediable, surface `degraded_reason.owning_page` so the user grants Editor access. `meta_video_source_unavailable` = unrecoverable, pick another ad.
- **Analyzer `ASSET_UNAVAILABLE`** → fallback failed too (no source URL, no thumbnail). Rare — pick another ad; if widespread on one account, `send_feedback` with IDs.
- **Analyzer `INVALID_INPUT: disallowed host`** → the URL analyzer's SSRF guard rejected the URL (localhost/private IP/non-http). Ask the user for a public URL.
