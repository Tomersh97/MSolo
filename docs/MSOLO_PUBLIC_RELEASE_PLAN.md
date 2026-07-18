# MSolo Public Release Plan

Single source of truth for the MSolo solo-first MapleStory v83 private server: product goals, architecture understanding, feature design, toolchain, phased delivery, and production launch.

---

## 1. What this project is

MSolo is a **solo-first** experience built on a Global MapleStory **v83** server, plus a branded client and a public website.

**High-level architecture (existing stack):**

- One Java 21 JVM: login (8484) + worlds/channels (7575+) via Netty
- Authoritative game state in memory (`Server` → `World` → `Channel` → `MapleMap`)
- MySQL persistence (accounts, characters, inventory, quests, shops, …)
- Static data from server-side WZ XML; dialogue/events from GraalJS scripts
- Original GMS v83 client (external) speaks the v83 packet protocol

This is a tightly coupled multiplayer monolith. Solo mode is a **policy + content** project on top of it, not a rewrite.

---

## 2. Locked product decisions

| Topic | Decision |
|---|---|
| Scale | Launch small (~20–50 concurrent); design ops so a spike toward ~300+ is absorbable without a rewrite |
| Isolation model | Shared overworld maps; chat allowed; **block** consequential multiplayer (trade, foreign drops, parties, guilds, player fame, merchants). Private instances for PQs/bosses/campaign |
| Monetization | Voting + donations → NX; Cash Shop cosmetics-focused; **no EXP coupons** (or similar power rate items); NX also drops from mobs (with anti-farm caps) |
| Hosting | No preferred cloud; use cost-effective VPS (e.g. Hetzner / OVHcloud) with a documented scale-up path |
| Code ownership | Rebranded **fork** of Cosmic + Cosmic-client — own the product/brand/content/ops; keep AGPL lineage honest (not a clean-room rewrite) |

### Ownership / AGPL (Cosmic server)

- Server Java + scripts: **AGPL-3.0**. Network use of modified AGPL requires offering corresponding source to players.
- Keep `LICENSE`, publish source (public Git and/or site “Source” page), retain attribution to OdinMS / HeavenMS / Cosmic.
- Client/WZ: derived from Nexon GMS v83 — rebrand packaging; never claim Nexon affiliation; residual legal risk of public private-server hosting is accepted.
- New separate tools (website, installer) can use MIT/Apache if they do not embed AGPL server code.

**Recommendation:** Own the *product*. Keep the *lineage* honest.

---

## 3. Original feature ideas (product requirements)

These are the starting requirements from product ideation:

### Solo focus despite multiplayer roots

Prefer isolating consequential interaction. If full outdoor isolation is not used, **chat is the only social interaction enabled**. Disable:

- Item transfer (trade / dropped items / FM player shops)
- Parties / guilds / alliances (PQs must be converted to solo — see below)
- Player fame / defame

### Rates

- Testing rates are not production rates
- Prefer level-dependent (and later campaign-aware) rates
- Start lower to push players toward campaign/quests
- Late game: grind becomes harder → grind rates more attractive
- **Quest EXP should generally be favored over grinding**
- Grinding still matters for essential drops

Config note from codebase review: `quest_rate` only applies when `USE_QUEST_RATE` is true; otherwise quest rewards follow normal player rates.

### Regional campaign quest chains

The game has **multiple regional campaigns**, one per major region (not a single world campaign). Each region gets its own campaign chain built from enjoyable existing town quest lines in that region.

Victoria Island is only an **example** of one regional campaign (e.g. town chapters such as Perion). Later regions (Ossyria and beyond) follow the same pattern with their own campaigns.

Completing parts of a regional campaign grants:

- Fast-travel within that region to associated towns
- Special rewards not typically found elsewhere
- Better EXP than ordinary quests
- Fame toward renown

### Renown system

Renown = character **fame** (not player-to-player fame).

Fame sources: quests that grant fame, special bosses, campaign completion. (Voting should grant **NX**, not renown power.)

Higher fame unlocks specialties (e.g. endgame renown store: Stormcaster gloves, Facestompers, master books, etc.).

### Items

- Master books / OP specialty gear via renown store (not Free Market dump)
- White/Chaos scrolls: prefer rare regional drops + pity/fragments + deterministic campaign/renown sources — **not** “every mob globally + 1-meso shop,” which collapses progression (baseline Cosmic already has global WS/CS drops; Agent M 1-meso stock was a temporary experiment to undo)

---

## 4. Architecture implications for solo

| Fact | Implication |
|---|---|
| `MapleMap` is shared multiplayer | Full per-player outdoor phasing is invasive; prefer shared maps + interaction gates |
| Social/economy opcodes are registered | Gate in Java handlers (server-side), not only by hiding client UI |
| PQs/expeditions assume parties | `USE_ENABLE_SOLO_EXPEDITIONS` only lowers expedition min size; PQs need script-by-script conversion |
| Class balance assumes party buffs/trade | Replace with consumables, blessings, solo-tuned encounters — not AI party members first |
| Content spans WZ + Java + JS | Campaign state should live in a Java/MySQL progression service; scripts orchestrate |

**Feasibility:** The requested features are implementable. Hardest early bets: solo PQ conversions and closed economy. Lowest early value: full map isolation and AI companions.

---

## 5. Feature design (consolidated)

### 5.1 Solo interaction policy

**Allowed:** map visibility, chat/whisper, buddy list (cosmetic), NPC shops/storage/Cash Shop, private event instances.

**Blocked:** player fame, parties/party search, guilds/alliances, trades/player shops/merchants, picking up other players’ drops, family (off), CPQ (off), Duey P2P if abused.

Message when blocked: `MSolo is a solo-progression server. That action is disabled.`

### 5.2 Rate philosophy

- Production start (tune in beta): modest grind EXP (e.g. 2×–4× early), higher quest feel via `USE_QUEST_RATE` + `quest_rate`, modest meso/drop
- Measure time-to milestones (first job / 30 / 70 / first regional campaign complete) — not isolated kill EXP
- Avoid 100× mesos and 1-meso BiS shops

### 5.3 Regional campaigns (multi-region system)

Campaigns are **per region**. The progression service and ledger must be campaign-id keyed (e.g. `victoria`, `ossyria`, …) so new regions plug in without redesign.

Shared template for every regional campaign:

- Ordered **town chapters** inside that region, using existing enjoyable quest chains
- Per chapter: strong quest EXP, renown fame, in-region fast-travel unlock, unique reward/recipe fragment
- A region **capstone** (solo-tuned)
- Ledger UI: which regional campaigns exist, current chapter, next objective, town checklist, unlocked travel, renown blurb

**Persistence:** MySQL tables keyed by `(character, campaign_id, chapter_id)` plus fast-travel keys; Java service; scripts call helpers only.

**Example — Victoria Island** (first campaign to ship; not the only one):

```
Victoria Island campaign (example)
├── Lith Harbor intro
├── Henesys / Ellinia / Perion / Kerning chapters
└── VI Capstone (solo-tuned)
```

**First vertical slice:** reusable campaign framework + Victoria Island (ledger + Henesys chapter + Henesys↔Lith travel + renown grant + one solo-friendly encounter). Later phases add further regional campaigns using the same template.

### 5.4 Renown tiers (example — tune in beta)

| Fame | Unlock |
|---|---|
| 0–9 | Baseline shops |
| 10 | Tier-1 renown shop |
| 25 | Tier-2 (books/recipes) |
| 50 | Tier-3 (master books subset) |
| 80 | Tier-4 specialty gear |
| 100+ | Cosmetics / titles / account legacy cosmetics |

NX must not buy fame/renown tiers.

### 5.5 Cash Shop / NX

**NX sources:** vote callbacks, donations (Stripe/PayPal), curated mob NX drops with daily cap.

**CS include:** cosmetics, non-power convenience.

**CS exclude:** EXP/meso/drop coupons that break rate philosophy; raw power equivalent to renown BiS.

### 5.6 Stretch systems (later)

- Solo PQ conversions (Kerning → Ludi → Orbis): party checks → solo; multi-person switches → sequential/timed
- Adaptive / practice bosses
- Monster journal / contracts (Monster Book extensions)
- Account legacy: shared travel knowledge + cosmetics vault; power stays per character
- Open-source helpers: campaign validator, economy simulator, party-gate script linter

---

## 6. Toolchain

### Core

- JDK 21 (Corretto/Temurin), Maven Wrapper, IntelliJ IDEA
- MySQL 8.4+, HeidiSQL/DBeaver
- Git + GitHub, Docker Desktop

### MapleStory-specific

- **HaRepacker-resurrected** (GMS old): edit client `.wz`, export Private Server XML → `MSolo/wz`
- **HaCreator** (optional) for map work
- Keep client WZ and server XML synchronized
- In-repo `tools/mapletools` for quest/drop/maker utilities

### Content / web / ops

- GraalJS NPC/quest/event/portal/reactor scripts; prefer NPC UIs for new systems
- Website: Next.js or Express marketing + account API sharing `accounts` (bcrypt-compatible)
- Stripe/PayPal donations; vote site callbacks; Cloudflare DNS/CDN
- VPS Ubuntu LTS; Docker Compose or systemd; Caddy/Nginx for web TLS; Netdata → Prometheus/Grafana; `mysqldump` + snapshots
- JUnit 5 + Mockito; Testcontainers for DB; load/smoke for login capacity

### Client packaging

- Fork Cosmic-client; rebrand; Inno Setup or launcher with WZ hash manifest; pin production DNS; distribute via CDN/Releases

---

## 7. Phased work plan

### Phase 0 — Foundations (2–3 weeks)

Design lock (sections 5.x), secrets via env (never commit DB passwords), Cosmic→MSolo branding, AGPL source policy on site, local golden path (MySQL → server → client → login), one world / few channels.

### Phase 1 — Easy features (learn the codebase)

Solo gates; `USE_QUEST_RATE` + intentional rates; quest-point→fame; rebuild Agent M (real prices, no 1-meso BiS); CS cosmetics pass; NX mob drops + caps; unit tests for policy.

### Phase 2 — Core solo progression

Progression service + Liquibase (multi-campaign ids); campaign ledger; ship **Victoria Island** as the first regional campaign (town arcs, fast travel, capstone); renown shops; integration tests. Framework must already support additional regions.

### Phase 3 — Stretch

Further **regional campaigns** (reuse VI template); solo PQs; adaptive bosses; journal/contracts; account legacy; helper tools + CI lint for party gates.

### Phase 4 — Website / NX pipeline

Register, vote, donate (idempotent receipts), admin audit, ToS/privacy/disclaimer, AGPL source link.

### Phase 5 — Client installer

Cosmic-client fork, rebrand, installer/launcher, WZ sync workflow documented and used.

### Phase 6 — Production

Small VPS compose (game + MySQL + web); backups + restore drill; monitoring; staging; CI (build/test/image); closed alpha → open beta → public; wipe policy decided before beta.

**Indicative calendar:** months 0–1 phases 0–1; 2–3 phase 2; 3–5 phases 3–5 in parallel; ~6 launch. Prefer content quality over calendar.

---

## 8. Production sketch

**Launch:** 4–8 vCPU, 16–32 GB RAM NVMe; ports 8484 + channel range; Cloudflare on web only.

**Growth:** vertical resize → dedicated MySQL → more channels (honest limit: still one JVM) → TCP DDoS options for game ports.

**CI/CD:** PR compile+tests; main JAR/Docker → GHCR; optional staging deploy; party-gate lint when scripts mature.

---

## 9. Testing strategy

| Layer | Focus |
|---|---|
| Unit | Solo policy, rates, renown thresholds, NX idempotency |
| Integration | Liquibase, shop gates, campaign progress, webhooks |
| Content | PQ conversion checklist; first regional campaign golden path (Victoria Island), then each new region |
| Load | Login storms; 50→200 concurrent |
| Security | Packet abuse on disabled features; NX dupes; web injection |

Rule: no custom solo feature merges without automated tests for its Java invariants.

---

## 10. Suggested repo layout (eventually)

- `MSolo/` — game server (AGPL Cosmic fork)
- `MSolo-client/` — client packaging
- `MSolo-web/` — marketing + accounts + vote/donate
- This file — plan (single document)
- Optional later: `tools/` open-sourced helpers

---

## 11. Acceptance criteria — first regional campaign slice (Victoria Island)

Victoria Island is the **first** regional campaign used to prove the multi-region system.

- Progression data model supports multiple `campaign_id`s (not hard-coded to Victoria only)
- New character always knows the next meaningful objective for the active regional campaign
- Every required upgrade has a deterministic solo source (not a 1-meso dump shop)
- No quest/encounter requires a second client
- Completing a town chapter unlocks travel or a service **in that region**
- Capstone tests mechanics, not only potions/DPS
- Early play visibly favors quest EXP over grind
- Two clients cannot trade, party, fame, or loot each other’s drops; chat still works
- Adding a second regional campaign later reuses the same ledger/service template without a rewrite
