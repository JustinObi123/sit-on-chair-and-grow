# 🪑 Sit on Chair and Grow

> A competitive Roblox multiplayer game where you collect orbs to grow your chair, then push rivals off theirs.

---

## 🎮 Core Concept

Sit on your chair. Collect glowing orbs to grow. Press **G** near a seated player to ragdoll them off — their chair resets to size 1. The player with the biggest chair when the timer runs out wins.

---

## ✅ What's Already Built

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
- [x] Flat 512×512 grass Baseplate map at Y=0
- [x] `Heartbeat` throttled to 20fps in ChairManager and OrbManager

---

## 🗺️ Roadmap

---

### Phase 1 — Lock the Core Loop
> Make existing mechanics feel intentional and bug-free before building on them.

- [ ] Finalize tunables (orb growth, magnet range, boost drain) through playtesting
- [ ] Define a hard size ceiling to prevent unbounded growth
- [ ] Decide push trade-off: does pusher inherit full size, half, or capped amount?
- [ ] Test edge cases: push during boost, simultaneous pushes, pushed at size 1

**Current tunables for reference:**
```
ORB_GROWTH    = 0.03      MAGNET_RANGE  = 12
ORB_COUNT     = 20        MIN_SPEED     = 2
COLLECT_RANGE = 2.5       MAGNET_SPEED  = 14
BASE_SPEED    = 18        TURN          = 2.6
LERP          = 6         BOOST_MULT    = 2.0
BOOST_DRAIN   = 0.15      BOOST_RATE    = 0.5
```

---

### Phase 2 — Lobby System & Map Voting
> Pre-game flow that brings players together, lets them vote, and transitions cleanly into a match.

- [ ] **Lobby area** — a dedicated waiting room/spawn zone separate from the game arena; players spawn here between rounds
- [ ] **Player count threshold** — game only starts once a minimum number of players are ready (e.g. 2+)
- [ ] **Map voting UI** — shows available maps with thumbnails/names and a live vote count; players click to vote
- [ ] **Game mode voting UI** — players vote on which mode to play alongside the map vote
- [ ] **Voting timer** — countdown visible to all; most-voted map + mode wins; ties broken randomly
- [ ] **Intermission countdown** — after voting closes, a short "Match starting in 3…2…1" screen before teleport
- [ ] **Auto-start fallback** — if player count threshold isn't met after a timeout, start anyway or loop intermission
- [ ] **Upcoming match display** — lobby shows the winning map + mode to all players while countdown runs

---

### Phase 3 — Game Modes
> Three core competitive formats. Each changes how winning is determined and how chairs interact.

---

#### 🏆 Mode 1: Timed Free-for-All *(default)*
> Every player for themselves. Biggest chair when the 5-minute timer hits zero wins.

- [ ] 5-minute server-authoritative round timer displayed on HUD
- [ ] Live size leaderboard visible to all players during the round
- [ ] Winner = player with the largest `TargetSize` when timer expires
- [ ] Round results screen: winner, top 3, your final size and placement
- [ ] Chairs phase through each other (current behaviour — no friendly collision needed)

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

- [ ] Round state machine: `Lobby → Voting → Countdown → Active → Results → Lobby`
- [ ] Mode-aware win condition logic (FFA size check vs. survival elimination vs. team sum)
- [ ] Per-round scoring tracked server-side: size reached, players pushed, time spent as #1
- [ ] Fresh chair at size 1 for every player at round start
- [ ] Clean teleport: lobby → map → lobby between rounds
- [ ] Spectator state for late-joining players mid-round

---

### Phase 5 — Maps
> All maps are flat. Variety comes from layout, theme, hazards, and edge design — not height.

**Map Design Rules:**
- Every map is a single flat plane — no ramps, no hills, no multi-level geometry
- Boundary walls or kill-planes prevent chairs from drifting off permanently
- Evenly distributed spawn points across the map
- Orb spawn zones defined per map (no orbs in dead corners)
- All maps support all three game modes

---

#### 🟩 Map 1: Grass Classic *(current)*
- [ ] Polish the existing Baseplate: add boundary walls, defined spawn points, orb zones
- [ ] Aesthetic pass: sky, lighting, simple decorative props around the edges

#### 🌌 Map 2: Neon Void
- [ ] Dark map with glowing floor tiles and a void border
- [ ] High-contrast aesthetic — chairs and orbs pop visually against the dark background
- [ ] Kill-plane border: fall off = instant chair reset

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

- [ ] **HUD:** current size, round timer, live rank, orb count, current mode indicator
- [ ] **Push feedback:** screen shake, impact VFX, sound on successful push, "Knocked off!" callout
- [ ] **Orb collection:** pop/sparkle on pickup, size-up flash, satisfying sound
- [ ] **Growth visuals:** glow that intensifies with size, trail that scales with chair
- [ ] **Boost feedback:** speed lines, trail VFX, audible whoosh, boost meter in HUD
- [ ] **Camera:** smooth zoom-out as chair grows, large chairs stay framed
- [ ] **Music:** lobby loop, intermission sting, per-mode active round music, round-end sting
- [ ] **Announcer text:** "New leader!", "10 seconds left!", "[Player] has been eliminated!", "Teams are tied!"

---

### Phase 7 — Progression & Economy
> Give players reasons to come back. This is where retention lives.

- [ ] **DataStore setup** — coins, unlocks, stats, level (build this early — hard to migrate later)
- [ ] **Soft currency (coins)** — earned from placement, pushes, playtime, mode-specific bonuses
- [ ] **Player XP / level** — climbs with playtime, gates cosmetic unlocks
- [ ] **Daily login reward / streak** — highest ROI retention feature on Roblox
- [ ] **Daily quests** — "push 10 players," "reach size 20," "win a Survival round"
- [ ] **Stats page** — total wins per mode, biggest chair ever, players pushed, win rate

---

### Phase 8 — Monetization
> Revenue without breaking fairness. Cosmetics and convenience, not pay-to-win.

- [ ] **Game Passes (permanent):** chair skins/trails, VIP tag, 2× coins multiplier, extra daily reward
- [ ] **Developer Products (consumable):** coin packs, one-round head-start size, extra boost charges
- [ ] **Roblox Premium Payouts:** give Premium members a perk to maximize engagement bonus
- [ ] ⚠️ Avoid anything that lets paying players permanently out-grow or become un-pushable

---

### Phase 9 — Social & Competitive
> Make the game sticky and shareable.

- [ ] **Global leaderboards** (DataStore-backed): biggest chair ever, most wins per mode, weekly resets
- [ ] **In-round live leaderboard** showing current size or team scores
- [ ] **Parties / friend join:** queue together, join friends' servers, see friends in lobby
- [ ] **Roblox Badges:** first win, size 50 reached, 100 pushes, first Survival win — free virality
- [ ] **Clip-worthy moments:** big push VFX dramatic enough to post/stream

---

### Phase 10 — Launch Readiness
> Don't get wrecked on day one.

- [ ] **Server-side size validation** — client-side scaling = exploitable; validate growth server-side ⚠️
- [ ] **Performance pass** — full server stress test (max players + max orbs + Heartbeat at 20fps)
- [ ] **Analytics** — session length, round completion rate, mode popularity, where players quit, conversion
- [ ] **Onboarding** — 10-second tutorial / first-time tooltips (sit, collect, press G, mode rules)
- [ ] **Mobile support** — on-screen boost button + push button (Roblox is mobile-heavy)
- [ ] **Game page polish** — icon, thumbnail, description, trailer GIF
- [ ] 📝 Remember: **File → Save to Roblox before every team test** (scripts wipe on team test)

---

## 🏗️ Architecture Reference

| Script | Location | Role |
|---|---|---|
| `ChairTemplate` | ServerStorage | VehicleSeat pivot, legs CanCollide=false |
| `ChairManager` | ServerScriptService | Orb spawning, push logic, server authority |
| `ChairMover` | LocalScript | Client movement, `ScaleTo`, network ownership, Y spring |
| `OrbManager` | ServerScriptService | Orb pool, magnet, collect logic |
| `Ragdoll` | ModuleScript | `PlatformStand` + `BallSocketConstraints` |
| `RagdollClient` | LocalScript (StarterCharacterScripts) | Client-side Humanoid state changes |
| `PushController` | LocalScript | G key push mechanic |

### Key Architecture Decisions
- **Chair scaling is client-side** — server sets `TargetSize` attribute, all clients lerp via `chair:ScaleTo()`
- **Character scaling is client-side** — server sets `CharScale` on HRP, all clients apply `character:ScaleTo()`
- **Server never calls `ScaleTo` during gameplay** — fixes the HipHeight unseating bug at size 1.45
- **`Occupied` attribute** updated every Heartbeat (not event-based) to survive brief SeatWeld disruptions
- **Survival mode collision ⚠️** — current Chairs collision group (chairs phase through each other) must be changed for Survival; chairs need to physically interact with each other in that mode

---

## 🔢 Suggested Build Order

1. **Phase 1** → lock the core loop feel
2. **Phase 2** → lobby + voting → now the game has a proper start/end flow
3. **Phase 3 + 4** → FFA mode first (simplest), then round structure → now it's a playable game
4. **Phase 6** → juice it → now it feels good
5. **Phase 5** → add maps + wire up map voting → now there's variety
6. **Phase 3 cont.** → add Survival + Team modes once FFA is solid
7. **Phase 7** → progression + DataStore → now people come back
8. **Phase 10** (anti-cheat + analytics) → pull these forward before any public release
9. **Phase 8 + 9** → monetization, social → growth and revenue

> ⚠️ Build DataStore (Phase 7) and server-side validation (Phase 10) earlier than their phase number suggests. Both are foundational and very painful to retrofit later.
>
> ⚠️ The Survival mode chair collision fix is a significant architecture change — plan it before any other Survival work begins.

---

*Maintained alongside development. Check off features as they ship.*
