# Quest system

## Overview
MSolo uses the classic MapleStory v83 quest model. Quest content is primarily defined in server-side WZ XML, Java validates and updates quest state, and optional JavaScript handles custom dialogue and behavior. The client renders quest text, NPC indicators, and the quest log from WZ data plus status packets sent by the server.

A character quest has three normal states: `NOT_STARTED`, `STARTED`, and `COMPLETED`. Per-character state also stores objective progress, completion and expiration times, completion/forfeit counts, and medal-map visits.

## General flow
1. **Availability:** `Quest.canStart` checks the current status and all start requirements, such as NPC, level, job, items, prerequisite quests, map, date, or repeat interval.
2. **Start:** `Quest.start` validates and runs WZ start actions, then `forceStart` creates a `STARTED` `QuestStatus`. Scripted quests normally call `forceStartQuest` from JavaScript and must perform custom start effects themselves.
3. **Progress:** mob kills update registered counters; item objectives are checked against inventory; scripts can update string-based progress; medal quests can track unique map visits.
4. **Turn-in:** `Quest.canComplete` requires `STARTED` status and validates every completion requirement.
5. **Completion:** `Quest.complete` checks reward capacity, marks the quest `COMPLETED`, runs WZ completion actions, and notifies the client. Scripted completions commonly use `forceCompleteQuest`, so their scripts must grant rewards explicitly.
6. **Follow-up:** a `nextQuest` action points the client toward the next quest, while that quest's own prerequisite requirements decide whether it is actually available.

Quests may also be forfeited, expire on a timer, restore a lost start item, or restart after an `interval` cooldown.

## Architecture and content files
- `wz/Quest.wz/QuestInfo.img.xml`: names, descriptions, categories, timers, auto flags, parent grouping, and medal metadata.
- `wz/Quest.wz/Check.img.xml`: start (`0`) and completion (`1`) requirements.
- `wz/Quest.wz/Act.img.xml`: start (`0`) and completion (`1`) actions and rewards.
- `wz/Quest.wz/Say.img.xml`: standard quest dialogue and requirement-failure text.
- `scripts/quest/<id>.js`: custom scripted quest dialogue and behavior; `medalQuest.js` is a generic medal fallback.
- `src/main/resources/db/tables/006-quest.sql`: persistent quest status, objective progress, and medal-map progress.

Normal quests use the WZ requirement/action pipeline. Scripted quests are selected by `startscript` or `endscript` requirements and run through GraalJS. Party quests are a separate event/instance subsystem and should not be confused with `Quest`/`QuestStatus`.

## Important implementation
- `server.quest.Quest`: loads and caches definitions; checks availability/completion; starts, completes, forfeits, and resets quests.
- `client.QuestStatus`: per-character state and mob/objective progress.
- `client.Character`: owns quest state, updates kill progress, sends client updates, manages expiration, and loads/saves quests.
- `server.quest.requirements.*`: requirement implementations.
- `server.quest.actions.*`: EXP, meso, item, fame, skill, buff, pet, progress, and follow-up actions.
- `QuestActionHandler`: handles normal/scripted start and completion, forfeit, and lost-item restoration packets.
- `QuestScriptManager` / `QuestActionManager`: load quest scripts and expose the `qm` scripting API.
- `RaiseUIStateHandler`: handles special `infoNumber`/item-UI quests.
- `tools.PacketCreator`: quest status, progress, timer, and completion packets.

## Rewards and configuration
Completion actions can grant or remove items, EXP, mesos, fame, skills, buffs, pet attributes, or other quest states. Item rewards support job/gender filtering, weighted random rewards, player-selected rewards, and temporary items. Inventory capacity is checked before a normal completion.

`config.yaml` controls dedicated quest EXP/meso rates, optional quest-points-to-fame behavior, repeatable-quest point rules, and the Temple of Time kill-count override.

## Future plans
The product plan intends to make quests central to solo progression:
- Favor quest EXP over early grinding using intentional quest rates.
- Build reusable regional campaigns, beginning with Victoria Island, from selected existing quest chains.
- Add a Java/MySQL campaign progression service and ledger keyed by region, with scripts acting as orchestration only.
- Reward campaign chapters with renown (character fame), regional fast travel, stronger EXP, and unique rewards; finish each region with a solo-tuned capstone.
- Convert selected party quests and encounters to solo-compatible content.
- Add automated tests for quest/campaign invariants and content validation.

See `docs/plan.md` for the authoritative product roadmap.

## Known caveats
- Script `forceStartQuest`/`forceCompleteQuest` changes state but does not run WZ actions.
- `start` date, `daybyday`, and `normalAutoStart` WZ requirements are not currently enforced by the Java requirement factory; end dates are enforced.
- Quest behavior has little automated coverage beyond script loading.
- Client WZ and server XML must remain synchronized when changing player-visible quest content.
