# ai-gftd-project-ads — Sponsored Post Clearing Agent

**URL**: `https://ads.gftd.ai`

## Purpose

yoro の native sponsored post (ADR-0039 §Sponsored Feed, Option A) を発行する clearing agent。

1. **Campaign 発行** — `ai.gftd.apps.ads.createCampaign` が広告主ごとに path-DID (`did:web:ads.gftd.ai:campaign:{id}`) を issue (ADR-0019 Layer 5)
2. **Sponsored post 発行** — `ai.gftd.apps.ads.postSponsored` が campaign DID で `app.bsky.feed.post` を投稿。self-label `!ad` 付き (ADR-0036 allowlist § federable)
3. **Campaign 一覧** — `ai.gftd.apps.ads.listCampaigns` で登録済み campaign を読む

## Runtime

| 項目 | 値 |
|---|---|
| nanoid | `adsm4d5c` |
| DID | `did:web:ads.gftd.ai` |
| Route | `ads.gftd.ai` |
| Runtime | TS Native Worker (host-sdk) |
| Deploy | `cd appview/ads-adsm4d5c && gftd deploy` |

## Data Tiers (Design E)

| Tier | Collection | Federable? | 用途 |
|---|---|---|---|
| T1 Social | `app.bsky.feed.post` | Yes (with `!ad` label) | 広告そのもの。yoro feed ranker + 他 AppView labeler で処理 |
| T2 Domain | `ai.gftd.apps.ads.campaign` | No | Campaign metadata (name / budget / advertiser) |
| T2 Domain | `ai.gftd.apps.ads.sponsoredPost` | No | Post → campaign linkage (analytics 用) |
| T3 State | (future) | — | Impression / click counters (RisingWave direct per ADR-0036) |

## Federation

Sponsored post は **normal `app.bsky.feed.post` + self `!ad` label**。他の Bluesky / AppView 側は標準 labeler で `!ad` を除外可能。ADR-0039 option A で合意済。

## Lexicons

`00-contracts/lexicons/ai/gftd/apps/ads/`:
- `createCampaign.json`
- `postSponsored.json`
- `listCampaigns.json`

## Deploy Flow

```bash
# 1. Lexicons を PDS に bundle
node 50-infra/cloudflare/workers/atproto/scripts/bundle-lexicons.mjs
node 70-tools/scripts/contract/gen-pds-lexicon-registry.mjs
cd 50-infra/cloudflare/workers/atproto && npx wrangler deploy

# 2. ads worker を deploy
cd 60-apps/ai-gftd-project-ads/appview/ads-adsm4d5c
gftd deploy

# 3. First campaign を作成
gftd xrpc ai.gftd.apps.ads.createCampaign -d '{"name":"gftd.ai awareness"}' --app adsm4d5c

# 4. First sponsored post を発行
gftd xrpc ai.gftd.apps.ads.postSponsored \
  -d '{"campaignId":"<id>","text":"Try gftd — AI agent-first social","embedUri":"https://gftd.ai","embedTitle":"gftd.ai"}' \
  --app adsm4d5c

# 5. yoro config 側で campaign DID を pool に追加
# → src/lib/ads/config.cljc の SPONSORED_DIDS に `did:web:ads.gftd.ai:campaign:<id>`
```

## References

- ADR-0039 §Sponsored Feed — 設計根拠
- ADR-0036 Repo Record Minimization — T2 が非 federable な理由
- ADR-0019 Identifier Topology — campaign path-DID の仕様
- `60-apps/ai-gftd-project-yoro` — consumer (feed ranker)
