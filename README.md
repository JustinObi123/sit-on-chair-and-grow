# 🪑 Sit on Chair and Grow

> A competitive Roblox multiplayer game where you collect orbs to grow your chair, then push rivals off theirs.

---

## 🎮 Core Concept

Sit on your chair. Collect glowing orbs to grow. Press **G** near a seated player to ragdoll them off — their chair resets to size 1. The player with the biggest chair when the timer runs out wins.

---

## ✅ What's Already Built

**Core loop**

- [x] Chair spawning from `ChairTemplate` (ServerStorage)
- [x] Orb collection with magnet range and collect range
- [x] Chair scaling — client-side via `ScaleTo()` in ChairMover (no unseating bug)
- [x] Character scaling — synced via `CharScale` attribute on HRP
- [x] Push mechanic — **G** key ragdolls victim, resets their chair to size 1, pusher inherits victim's size
- [x] Ragdoll system — `PlatformStand` + `BallSocketConstraints`, client-side state replication
- [x] Boost/sprint — hold Shift for 2× speed, drains 0.15 size per 0.5s
- [x] Trail orbs — pooled (8 pre-created), low growth value (0.01), 4s pickup cooldown
- [x] Multiplayer sync — `TargetSize` + `CharScale` attributes, all clients lerp
- [x] Network ownership — granted to sitting player via `seat:SetNetworkOwner(p)`
- [x] 1-second spawn protection on sit (blocks push and being pushed)
- [x] Chair collision group — chairs phase through each other, collide with floor
- [x] Flat 600×600 grass Baseplate map at Y=0
- [x] `Heartbeat` throttled to 20fps in ChairManager and OrbManager
- [x] Hard size ceiling — `MAX_SIZE = 50` prevents unbounded growth
- [x] Raycast Y-spring — chairs sit at the correct height on elevated maps, not just Y=0

**Match flow (round system)**

- [x] Full state machine — `Lobby → Voting → Countdown → Active → Results → Lobby` (`RoundManager`), auto-restarts on error
- [x] Lobby — isolated island far from the maps; players teleport in/out between rounds
- [x] Map + mode voting UI with live vote tally, vote timer, and auto-start fallback (`RoundClient`)
- [x] Intermission countdown ("Match starting in 5…") before teleport into the arena
- [x] Timed FFA win condition — 5-minute server timer, winner = largest `TargetSize`, results screen
- [x] Fresh size-1 chair per player at round start; chairs parked underground during lobby; auto-sit on round start
- [x] Late-join state sync — joiners receive the current round state so their UI is correct
- [x] `RoundSignals` module — shared BindableEvents coordinate chairs, orbs, and the kill-plane with round state

**Content & performance**

- [x] Second map — **Neon Void**: raised neon platform with a glowing floor, border walls, and a kill-plane
- [x] Per-map arena spawn points and orb spawn zones; orbs scatter into the active map only
- [x] Orb system scaled to **1000 orbs** with a heavy optimization pass — Lua-side caches, one Workspace scan per tick, squared-distance math, XZ-only magnetism, loop early-outs when no round is active

---

## 🗺️ Roadmap

---

### Phase 0 — Standards & Infrastructure

> Set the rules before writing a single line of new code. Every system built after this phase must comply with these standards.

#### 🖥️ Server Size

- [ ] **Max players: 12** — optimal balance of chaos, physics load, and network sync. Drop to 10 if performance testing reveals issues. Never exceed 16.
- [ ] Set `MaxPlayers = 12` in the game settings before any team testing *(verify in Studio — not confirmed set)*

#### 🎯 Performance Standards *(all new systems must meet these)*

- [ ] **FPS targets** — 60fps on mid-range PC, 30fps on mid-range mobile (e.g. iPhone 11 / mid-tier Android); define before building UI or VFX
- [ ] **Heartbeat throttle rule** — every new RunService connection must be throttled; no system may run faster than 20fps on the server; document the tick rate for each system *(currently honored — both managers run at 20fps)*
- [ ] **Remote event budget** — max 3 RemoteEvents firing per player per second under normal gameplay; all new events must be justified against this budget; consolidate where possible
- [ ] **DataStore write strategy** — writes are debounced (save on leave + every 5 min, never on every event); a write queue handles retries on failure; no raw `SetAsync` without pcall + retry logic
- [ ] **Part count budget** — max 500 active parts in Workspace during a live round (chairs + orbs + map + characters); each map must be verified against this before shipping
  - ⚠️ **Conflict:** `OrbManager` currently spawns **1000 orbs**, which alone is 2× this budget before chairs, characters, and map geometry are counted. Either raise the budget to match the optimized orb system or lower `ORB_COUNT`.
- [ ] **Memory cleanup contract** — every system that creates connections, parts, or GUI elements must have a corresponding cleanup function; cleanup is called on player leave, round end, and map change

#### 🧪 Testing Standards *(every phase must pass these before moving on)*

- [ ] **Phase completion criteria** — each phase has a written pass/fail checklist; a phase is not "done" until all items pass in a team test
- [ ] **Test progression order** — solo → 2-player → 6-player → 12-player (full server); never skip stages
- [ ] **Regression rule** — when a new phase is added, re-run the previous phase's test checklist to confirm nothing broke
- [ ] **Save before every team test** — File → Save to Roblox; scripts wipe on team test without this 📝

#### 🔧 Dev Workflow

- [ ] All scripts saved to GitHub after each major session (copy/paste from Studio) *(repo currently holds README only — scripts not yet committed)*
- [ ] README checked off as features ship *(in progress — this sync)*
- [ ] Bugs logged as GitHub Issues with reproduction steps

---

### Phase 1 — Lock the Core Loop

> Make existing mechanics feel intentional and bug-free before building on them.

- [ ] Finalize tunables (orb growth, magnet range, boost drain) through playtesting
- [x] Define a hard size ceiling to prevent unbounded growth *(`MAX_SIZE = 50`)*
- [x] Decide push trade-off: does pusher inherit full size, half, or capped amount? *(decided: pusher inherits the victim's full size)*
- [ ] Test edge cases: push during boost, simultaneous pushes, pushed at size 1

**Current tunables for reference:**

```
-- Orbs (OrbManager, server)
ORB_GROWTH    = 0.03       MAGNET_RANGE  = 12
ORB_COUNT     = 1000       COLLECT_RANGE = 2.5
ORB_SIZE      = 1.2        MAGNET_SPEED  = 14
MAP_HALF      = 240        MIN_SPEED     = 2
RESPAWN_T     = 3
Trail orbs: pool of 8 · size 0.7 · growth 0.01 · 4s pickup cooldown

-- Movement & boost (ChairMover, client)
BASE_SPEED    = 18         BOOST_MULT    = 2.0
TURN          = 2.6        BOOST_DRAIN   = 0.15
LERP          = 6          BOOST_RATE    = 0.5
SEAT_H        = 1.75       BOOST_MIN     = 1.05
(speed scales as BASE_SPEED / sqrt(size) — bigger chairs move slower)

-- Push, ragdoll & size (ChairManager, server)
MAX_SIZE      = 50         PCDWN = 2     (push cooldown, seconds)
PFORCE        = 55         RTIME = 2     (ragdoll duration, seconds)
BOOST_MIN     = 1.05       FLIP  = 0.1 / FTIME = 1.5  (flip-recovery)
```

---

### Phase 2 — Lobby System & Map Voting

> Pre-game flow that brings players together, lets them vote, and transitions cleanly into a match.

- [x] **Lobby area** — a dedicated waiting room/spawn zone separate from the game arena; players spawn here between rounds
- [x] **Player count threshold** — game only starts once a minimum number of players are ready *(mechanism in place; `MIN_PLAYERS` currently 1 for testing — raise before launch)*
- [x] **Map voting UI** — shows available maps with names and a live vote count; players click to vote *(thumbnails TBD)*
- [x] **Game mode voting UI** — players vote on which mode to play alongside the map vote *(UI built; only FFA exists so far)*
- [x] **Voting timer** — countdown visible to all; most-voted map + mode wins; ties broken by list order (deterministic, no randomness)
- [x] **Intermission countdown** — after voting closes, a short "Match starting in 5…4…3" screen before teleport
- [x] **Auto-start fallback** — if player count threshold isn't met after a timeout (`LOBBY_TIMEOUT = 60s`), start anyway
- [ ] **Upcoming match display** — lobby shows the winning map + mode to all players while countdown runs *(countdown screen exists but currently shows "Get ready!" rather than the chosen map name — stubbed)*

---

### Phase 3 — Game Modes

> Three core competitive formats. Each changes how winning is determined and how chairs interact.

---

#### 🏆 Mode 1: Timed Free-for-All *(default — built)*

> Every player for themselves. Biggest chair when the 5-minute timer hits zero wins.

- [x] 5-minute server-authoritative round timer displayed on HUD
- [ ] Live size leaderboard visible to all players during the round *(basic size shown via default `leaderstats` list; dedicated panel is Phase 7)*
- [x] Winner = player with the largest `TargetSize` when timer expires
- [ ] Round results screen: winner, top 3, your final size and placement *(winner + size shown; top 3 and personal placement pending)*
- [x] Chairs phase through each other (current behaviour — no friendly collision needed)

---

#### ☠️ Mode 2: Survival

> Last chair growing wins. Get pushed off too many times and you're eliminated.

- [ ] **Lives system** — each player starts with a set number of lives (e.g. 3); losing your chair costs 1 life
- [ ] **Elimination** — reach 0 lives → chair disappears, player enters spectator mode
- [ ] **Chair collision fix ⚠️** — currently chairs phase through each other due to the Chairs collision group; Survival requires chairs to physically collide and push each other, not just the G-key mechanic. Needs a new collision setup (chairs collide with each other AND the floor, not just floor)
- [ ] Win condition: last player with lives remaining
- [ ] **Sudden death** — when only 2 players remain, orb spawn rate doubles and the arena shrinks (or a timer triggers forced confrontation)
- [ ] **Lives indicator** — small chair icons above each player's head showing remaining lives
- [ ] Spectator cam for eliminated players that follows remaining competitors

---

#### 👥 Mode 3: Team Mode

> Teams compete. The team with the highest combined chair size at the end of 5 minutes wins.

- [ ] Auto-team assignment on round start (teams balanced by player count)
- [ ] Team color coding — chair color reflects team assignment (e.g. red vs blue)
- [ ] **Team score display** — combined `TargetSize` of all living teammates shown live on HUD
- [ ] Friendly fire rule — players cannot push teammates off (G-key blocked on same team)
- [ ] Round results: winning team name, MVP (largest individual chair), all team scores
- [ ] Team indicator above player heads (color tag or team icon)

---

#### 🔮 Future Modes *(ideas to flesh out later)*

- [ ] TBD — more modes to be designed as the game grows

---

### Phase 4 — Round Structure & Scoring

> The server-side skeleton that powers all three modes.

- [x] Round state machine: `Lobby → Voting → Countdown → Active → Results → Lobby`
- [ ] Mode-aware win condition logic (FFA size check vs. survival elimination vs. team sum) *(FFA only)*
- [ ] Per-round scoring tracked server-side: size reached, players pushed, time spent as #1
- [x] Fresh chair at size 1 for every player at round start
- [x] Clean teleport: lobby → map → lobby between rounds
- [ ] Spectator state for late-joining players mid-round *(UI state syncs for late joiners; true spectator mode not built — they currently spawn into the live round)*

---

### Phase 5 — Maps

> All maps are flat. Variety comes from layout, theme, hazards, and edge design — not height.

**Map Design Rules:**

- Every map is a single flat plane — no ramps, no hills, no multi-level geometry
- Boundary walls or kill-planes prevent chairs from drifting off permanently
- Evenly distributed spawn points across the map
- Orb spawn zones defined per map (no orbs in dead corners)
- All maps support all three game modes
- Part count verified against Phase 0 budget (≤500 total active parts) before shipping

---

#### 🟩 Map 1: Grass Classic *(current)*

- [x] Polish the existing Baseplate: add boundary walls, defined spawn points, orb zones *(arena walls, per-map spawn points, and orb spawn radius in place)*
- [x] Aesthetic pass: sky, lighting, simple decorative props around the edges *(decorative trees/props around the border added; sky/lighting can still be refined)*

#### 🌌 Map 2: Neon Void *(built)*

- [x] Dark map with glowing floor tiles and a void border *(`NeonFloor` + `CWall` border ring)*
- [x] High-contrast aesthetic — chairs and orbs pop visually against the dark background
- [x] Kill-plane border: fall off = teleported back to a spawn point ⚠️ *(differs from original "instant chair reset" intent — currently repositions the player, does not reset chair size; decide which behavior you want)*

#### 🏙️ Map 3: Grid City

- [ ] Urban grid layout with low divider walls (chairs can ride along or around them)
- [ ] Creates natural lanes and choke points without breaking the flat rule
- [ ] Good for team mode — natural territory splitting

#### 🧊 Map 4: Ice Rink

- [ ] White/blue icy flat surface
- [ ] Surface friction modifier: chairs drift and slide further after movement input
- [ ] Pushes feel more chaotic and rewarding on this map

#### ☁️ Map 5: Sky Platform

- [ ] Floating platform in the sky, smaller than the standard map
- [ ] All four edges are open — getting pushed near the edge is very high stakes
- [ ] Good pairing with Survival mode

#### ⬛ Map 6: Tiny Arena

- [ ] Very small flat platform — forces constant confrontation
- [ ] Designed for shorter, chaotic rounds
- [ ] Works best with Survival mode

#### 🔮 More Maps *(future)*

- [ ] Additional maps to be designed and added as the game grows

---

### Phase 6 — Game Feel & Juice

> Make actions feel satisfying. This is what separates good Roblox games from great ones.

- [ ] **Push feedback:** screen shake, impact VFX, sound on successful push, "Knocked off!" callout
- [ ] **Orb collection:** pop/sparkle on pickup, size-up flash, satisfying sound
- [ ] **Growth visuals:** glow that intensifies with size, trail that scales with chair
- [ ] **Boost feedback:** speed lines, trail VFX, audible whoosh, boost meter on screen
- [ ] **Camera:** smooth zoom-out as chair grows, large chairs stay framed
- [ ] **Music:** lobby loop, intermission sting, per-mode active round music, round-end sting
- [ ] **Announcer text:** "New leader!", "10 seconds left!", "[Player] has been eliminated!", "Teams are tied!"
- [ ] **Round transition animations:** fade in/out, results screen slide-in, trophy animation for winner

---

### Phase 7 — In-Game UI, Shop & Power-Ups

> The full UI layer players interact with every session — HUD, economy displays, shop, spin, daily rewards, and in-game power-up buttons.

---

#### 💰 Currency & HUD Display

- [ ] **Coin display** — top-left corner showing coin icon + formatted amount (e.g. "12.22K"); "+" button to open purchase screen
- [ ] **Gem/premium currency display** — secondary currency shown alongside coins
- [ ] **Animated coin counter** — numbers tick up when coins are earned mid-round
- [x] **Round timer** — prominent countdown visible during active rounds
- [ ] **Current size indicator** — your live chair size displayed clearly on HUD *(only via default leaderstats today)*
- [ ] **Rank indicator** — your live placement during the round (e.g. "#2 of 8")

#### 🏆 In-Game Leaderboard

- [ ] **Top-right leaderboard panel** — shows top 4 players with avatar thumbnail, username, and size/score
- [ ] **Expand button** — arrow or toggle to see full server leaderboard
- [ ] **Your row highlighted** — if you're outside top 4, your row is pinned and highlighted at the bottom
- [ ] **Real-time updates** — event-driven, not loop-driven; fires on size change only, throttled to max 2fps to protect performance
- [ ] **Mode-aware display** — shows size in FFA, lives remaining in Survival, team scores in Team Mode

#### 🛍️ Shop

- [ ] **Shop button** — bottom-left corner with icon; opens full-screen shop overlay
- [ ] **Chair skins tab** — browse, preview, and buy chair cosmetics (coin or Robux priced)
- [ ] **Chair trails tab** — browse, preview, and buy trail effects
- [ ] **Equipped indicator** — clear "Equipped" badge on currently active items
- [ ] **Categories** — Free, Coins, Robux, Limited/Event
- [ ] **Preview mode** — clicking a skin/trail previews it on your chair in real time before buying

#### 🎰 Spin Wheel

- [ ] **Spin button** — bottom-left area with notification badge when a free spin is available
- [ ] **Animated spin wheel** — satisfying spin animation with coin, gem, and cosmetic prize slots
- [ ] **Free spin timer** — e.g. 1 free spin every 4 hours; timer shown on button
- [ ] **Multi-spin** — spend coins or Robux for extra spins in one session
- [ ] **Prize reveal animation** — dramatic pop-up when landing on a rare item

#### 📅 Daily Gift

- [ ] **Daily button** — bottom-left with red notification badge when unclaimed
- [ ] **Floating gift icon** — animated gift floats on screen at session start when daily is ready to claim
- [ ] **"CLAIM!" button** — prominent top-center button when daily reward is available
- [ ] **Login streak calendar** — shows the current streak and upcoming rewards for each consecutive day
- [ ] **Claim animation** — coins/items burst out with satisfying sound on claim

#### ⚡ In-Game Power-Up Buttons

- [ ] **Bottom-center action bar** — 3 large buttons visible during active rounds
- [ ] **Push All** — instantly ragdolls all players within a radius off their chairs; cooldown or coin cost; big satisfying VFX
- [ ] **2x Speed** — temporary 2× movement speed boost for a short duration; cooldown-based
- [ ] **2x Size** — temporary size multiplier for a short duration; cooldown or coin cost
- [ ] **Cooldown UI** — each button shows a visual cooldown ring/timer so players know when it's ready again
- [ ] **Mobile-friendly** — buttons large enough to tap comfortably on phone screens

#### 🎁 VIP & Event UI

- [ ] **VIP zone** — special area on maps for VIP pass holders (visual indicator above VIP players' chairs)
- [ ] **Event banners** — seasonal event announcements shown in lobby and on HUD
- [ ] **Limited-time offer popups** — timed shop deals shown tastefully (not intrusive)

---

### Phase 8 — Progression & Economy

> Give players reasons to come back. This is where retention lives.

- [ ] **DataStore setup** — coins, gems, unlocks, stats, level; write queue + pcall + retry on every save; debounced writes (see Phase 0 standards)
- [ ] **Soft currency (coins)** — earned from placement, pushes, playtime, power-up use, mode-specific bonuses
- [ ] **Premium currency (gems)** — earned slowly in-game or purchased; used for spins and premium shop items
- [ ] **Player XP / level** — climbs with playtime, gates cosmetic unlocks, shown on profile
- [ ] **Daily login reward / streak** — highest ROI retention feature on Roblox
- [ ] **Daily quests** — "push 10 players," "reach size 20," "win a Survival round," "use Push All 3 times"
- [ ] **Stats page** — total wins per mode, biggest chair ever, players pushed, power-ups used, win rate

---

### Phase 9 — Monetization

> Revenue without breaking fairness. Cosmetics and convenience, not pay-to-win.

- [ ] **Game Passes (permanent):** chair skin bundle, trail bundle, VIP zone access, 2× coins multiplier, extra daily spin
- [ ] **Developer Products (consumable):** coin packs, gem packs, instant power-up charges, bonus spin bundle
- [ ] **Robux shop items:** exclusive chair skins, exclusive trails, limited event cosmetics
- [ ] **Roblox Premium Payouts:** give Premium members a daily gem bonus to maximize engagement revenue
- [ ] ⚠️ Avoid anything that lets paying players permanently out-grow or become un-pushable

---

### Phase 10 — Social & Competitive

> Make the game sticky and shareable.

- [ ] **Global leaderboards** (DataStore-backed): biggest chair ever, most wins per mode, most pushes, weekly resets
- [ ] **In-round live leaderboard** showing current size or team scores (see Phase 7)
- [ ] **Parties / friend join:** queue together, join friends' servers, see friends in lobby
- [ ] **Roblox Badges:** first win, size 50 reached, 100 pushes, first Survival win — free virality
- [ ] **Clip-worthy moments:** Push All VFX and big push animations dramatic enough to post/stream

---

### Phase 11 — Launch Readiness

> Don't get wrecked on day one.

- [ ] **Server-side size validation** — client-side scaling = exploitable; validate growth server-side ⚠️
- [ ] **Performance pass** — full 12-player server stress test (max orbs + all UI active + Push All triggered)
- [ ] **Analytics** — session length, round completion rate, mode popularity, shop conversion, where players quit
- [ ] **Onboarding** — 10-second tutorial / first-time tooltips (sit, collect, press G, how power-ups work)
- [ ] **Mobile support** — all HUD buttons and power-up bar verified on actual iOS and Android devices
- [ ] **Game page polish** — icon, thumbnail, description, trailer GIF showing Push All moment
- [ ] 📝 Remember: **File → Save to Roblox before every team test** (scripts wipe on team test)

---

### Phase 12 — Performance & Testing (Final QA)

> The comprehensive QA pass before public launch. Every item below must pass at 12 players before the game goes live.

---

#### ⚡ Performance Audit

- [ ] **Heartbeat audit** — list every RunService connection in the codebase; verify each is throttled per Phase 0 standards; remove or consolidate any unthrottled loops
- [ ] **Remote event audit** — log all RemoteEvents/RemoteFunctions; verify total firing rate is within Phase 0 budget at 12 players; merge redundant events
- [ ] **DataStore audit** — verify all saves go through the write queue; simulate 12 players finishing a round simultaneously and confirm no data loss or rate-limit errors
- [ ] **Memory leak audit** — join → play a full round → leave; run Roblox's MicroProfiler and confirm no growing memory usage over 5 consecutive rounds
- [ ] **Ragdoll cleanup verification** — trigger ragdoll on all 12 players simultaneously; verify all BallSocketConstraints are destroyed when ragdoll ends; check for orphaned constraints on player leave
- [ ] **UI refresh rate verification** — confirm leaderboard and HUD updates are event-driven or throttled; verify no UI element is connected to an unthrottled RunService loop
- [ ] **Chair physics budget** — test at max chair size with 12 players in Survival (worst-case collision load); verify server FPS stays above 20
- [ ] **Part count verification** — run each map at 12 players and count active Workspace parts; must stay under Phase 0 budget of 500 *(see Phase 0 orb-count conflict)*

---

#### 🧪 Functional Testing

**Solo tests (1 player):**

- [ ] Game starts alone without crashing or getting stuck in lobby
- [ ] Can collect orbs, grow chair, boost, use power-ups alone
- [ ] Round completes and returns to lobby without errors

**2-player tests:**

- [ ] Push mechanic syncs correctly — victim ragdolls on both clients simultaneously
- [ ] Size transfer is accurate — pusher receives correct size
- [ ] Both players see the same leaderboard ranking
- [ ] Boost drain visible to both players

**Full server tests (12 players):**

- [ ] All chairs spawn without overlap or collision issues at round start
- [ ] Leaderboard stays accurate under rapid size changes
- [ ] Push All triggers correctly and affects all players within radius
- [ ] Round timer stays in sync across all 12 clients
- [ ] Results screen shows correct winner

**Edge case tests:**

- [ ] **Player leaves while sitting** — chair cleans up, round continues
- [ ] **Player leaves while ragdolled** — BallSocketConstraints cleaned up, no server error
- [ ] **Player leaves as last alive in Survival** — round ends correctly, no infinite loop
- [ ] **Player leaves during Countdown** — game starts correctly with remaining players
- [ ] **Player joins mid-round** — enters spectator correctly, no chair spawned mid-round *(currently spawns into the live round — see Phase 4)*
- [ ] **0 players vote in lobby** — default map + mode selected (first in list), game starts
- [ ] **All players on same team** — Team Mode handles degenerate case without crash
- [ ] **Two players push each other simultaneously** — no duplicate ragdoll, no size duplication
- [ ] **Two players collect the same orb at the same frame** — no double-growth exploit
- [ ] **Push All fires while target is already ragdolled** — no error, no duplicate constraint

**Mode transition tests:**

- [ ] FFA → Survival → Team Mode → FFA: full cycle without state bleed between rounds
- [ ] State machine cannot get stuck: test every transition manually (Lobby→Voting, Voting→Countdown, Countdown→Active, Active→Results, Results→Lobby)
- [ ] Map transition: 12 players teleport from lobby to map simultaneously with no race conditions

**DataStore failure test:**

- [ ] Simulate DataStore unavailability (or test in Studio where DS may be limited) — game must degrade gracefully (warn player, don't crash, retry on next save interval)

**Mobile device test:**

- [ ] Test on actual iOS device — all HUD buttons tappable, FPS stays at or above 30
- [ ] Test on actual Android device — power-up bar, shop, spin, daily all functional
- [ ] On-screen controls don't overlap or obstruct gameplay view

---

#### 🔒 Security & Exploiter Audit

- [ ] **Size inflation test** — fire a RemoteEvent manually to attempt fake size growth; server-side validation must reject it
- [ ] **Push spoofing test** — attempt to trigger push on yourself or non-adjacent players via remote; server must validate proximity and eligibility
- [ ] **Power-up spam test** — attempt to fire Push All without cooldown via remote; server must enforce cooldown independently
- [ ] **DataStore injection test** — attempt to write arbitrary data via client-side exploit; server must only accept whitelisted values
- [ ] **Coin duplication test** — attempt to trigger multiple coin rewards for a single event; server must be authoritative on all economy transactions

---

## 🏗️ Architecture Reference

| Script           | Location                                    | Role                                                                 |
| ---------------- | ------------------------------------------- | -------------------------------------------------------------------- |
| `ChairTemplate`  | ServerStorage                               | VehicleSeat pivot, legs CanCollide=false                             |
| `ChairManager`   | ServerScriptService                         | Chair lifecycle, push/grow/boost logic, round chair placement, server authority |
| `ChairMover`     | LocalScript (StarterPlayerScripts)          | Client movement, `ScaleTo`, network ownership, raycast Y-spring      |
| `OrbManager`     | ServerScriptService                         | Orb pool (1000), magnet + collect logic, trail orbs                  |
| `Ragdoll`        | ModuleScript (ServerScriptService)          | `PlatformStand` + `BallSocketConstraints`                            |
| `RagdollClient`  | LocalScript (StarterCharacterScripts)       | Client-side Humanoid state changes                                   |
| `PushController` | LocalScript (StarterPlayerScripts)          | G key push mechanic                                                  |
| `RoundManager`   | ServerScriptService                         | Match state machine, vote tally, teleports, win detection           |
| `RoundSignals`   | ModuleScript (ServerScriptService)          | Shared BindableEvents (`RoundActivate` / `RoundDeactivate`)          |
| `RoundClient`    | LocalScript (StarterPlayerScripts)          | Lobby / voting / countdown / timer / results HUD                    |
| `KillPlane`      | Workspace.Map2_NeonVoid.KillPlane (Script)  | Returns players who fall off the Neon Void platform                  |

### Key Architecture Decisions

- **Chair scaling is client-side** — server sets `TargetSize` attribute, all clients lerp via `chair:ScaleTo()`
- **Character scaling is client-side** — server sets `CharScale` on HRP, all clients apply `character:ScaleTo()`
- **Server never calls `ScaleTo` during gameplay** — fixes the HipHeight unseating bug at size 1.45
- **`Occupied` attribute** updated every Heartbeat (not event-based) to survive brief SeatWeld disruptions
- **Round coordination via `RoundSignals`** — a ModuleScript holding shared BindableEvents; `ChairManager`, `OrbManager`, and the Neon Void kill-plane all react to `RoundActivate(mapId)` and `RoundDeactivate`
- **Per-map coordinates** — Grass Classic sits at the origin, Neon Void is a raised platform at ~(1000, 150, 0); chairs and orbs spawn into the active map's arena only, and the lobby is an isolated island far from both
- **Raycast Y-spring** — `ChairMover` raycasts straight down to find the real floor, so chairs sit at the correct height on elevated maps instead of assuming Y=0
- **Orbs parked underground (Y=-500) during lobby** — and the orb Heartbeat loop early-outs entirely when no round is active
- **Survival mode collision ⚠️** — current Chairs collision group (chairs phase through each other) must be changed for Survival; chairs need to physically interact with each other in that mode
- **Max server size: 12 players** — physics, sync, and DataStore load all tuned to this cap

---

## 🔢 Suggested Build Order

**Done so far:** core loop · lobby + voting (Phase 2) · round structure (Phase 4) · Timed FFA (Phase 3 Mode 1) · two maps (Grass Classic + Neon Void). **Next logical steps:** Phase 1 final polish → Phase 6 juice → Survival/Team modes (Phase 3) → more maps (Phase 5).

1. **Phase 0** → set standards, server size, dev workflow — do this before touching any code
2. **Phase 1** → lock the core loop feel
3. **Phase 2** → lobby + voting → proper start/end flow ✅
4. **Phase 3 + 4** → FFA mode first, then round structure → playable game ✅ *(FFA + structure done; Survival/Team remain)*
5. **Phase 6** → juice it → feels good
6. **Phase 5** → add maps + wire up map voting → variety *(2 of 6 maps done)*
7. **Phase 7** → full UI layer: HUD, shop, spin, daily, power-ups → looks like a real game
8. **Phase 3 cont.** → add Survival + Team modes once FFA is solid
9. **Phase 8** → progression + DataStore → people come back
10. **Phase 11** (anti-cheat + analytics) → pull these forward before any public release
11. **Phase 9 + 10** → monetization, social → growth and revenue
12. **Phase 12** → full QA pass → ship it

> ⚠️ Build DataStore (Phase 8) and server-side validation (Phase 11) earlier than their phase number suggests — both are foundational and painful to retrofit.
>
> ⚠️ The Survival mode chair collision fix is a significant architecture change — plan it before any other Survival work begins.
>
> ⚠️ Phase 12 is not optional — every item in the exploiter audit must pass before going public.

---

*Maintained alongside development. Check off features as they ship.*

*Last synced with the Studio build: 2026-05-31 — checked off lobby + voting, round structure, Timed FFA, and Neon Void; corrected the tunables block (notably `ORB_COUNT` 20 → 1000) and flagged the part-count budget conflict, the deterministic vote tiebreak, the kill-plane teleport-vs-reset behavior, and the stubbed upcoming-match display; resized the Grass Classic baseplate from 1500×1500 to 600×600 to sit close to the tree border.*
