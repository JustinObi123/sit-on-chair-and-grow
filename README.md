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

### Phase 2 — Make It a Game (Rounds & Objectives)
> Add the structure that turns the sandbox into a match.

**Win condition: Biggest chair at the buzzer** *(timed rounds, ~3–5 min)*

- [ ] Server-authoritative round timer
- [ ] Round state machine: `Intermission → Countdown → Active → Results → Repeat`
- [ ] Win condition logic (largest chair when timer hits 0)
- [ ] Round results screen (winner, top 3, your placement, size reached)
- [ ] Per-round scoring (size reached, players pushed, time spent as #1)
- [ ] Fresh chair at size 1 for every player each round
- [ ] Spectator state for players who join mid-round

---

### Phase 3 — Game Feel & Juice
> Make actions feel satisfying. This is what separates good Roblox games from great ones.

- [ ] **HUD:** current size, round timer, live rank, orb count
- [ ] **Push feedback:** screen shake, impact VFX, sound on successful push, "Knocked off!" callout
- [ ] **Orb collection:** pop/sparkle on pickup, size-up flash, satisfying sound
- [ ] **Growth visuals:** glow that intensifies with size, trail that scales with chair
- [ ] **Boost feedback:** speed lines, trail VFX, audible whoosh, boost meter in HUD
- [ ] **Camera:** smooth zoom-out as chair grows, large chairs stay framed
- [ ] **Music:** intermission loop vs. active round loop, sting on round end
- [ ] **Announcer text:** "New leader!", "10 seconds left!", "[Player] is on a rampage!"

---

### Phase 4 — Progression & Economy
> Give players reasons to come back. This is where retention lives.

- [ ] **DataStore setup** — coins, unlocks, stats, level (build this early — hard to migrate later)
- [ ] **Soft currency (coins)** — earned from placement, pushes, playtime
- [ ] **Player XP / level** — gates cosmetic unlocks
- [ ] **Daily login reward / streak** — highest ROI retention feature on Roblox
- [ ] **Daily quests** — "push 10 players," "reach size 20," "win a round"
- [ ] **Stats page** — total wins, biggest chair ever, players pushed, win rate

---

### Phase 5 — Monetization
> Revenue without breaking fairness. Cosmetics and convenience, not pay-to-win.

- [ ] **Game Passes (permanent):** chair skins/trails, VIP tag, 2× coins multiplier, extra daily reward
- [ ] **Developer Products (consumable):** coin packs, one-round head-start size, extra boost charges
- [ ] **Roblox Premium Payouts:** give Premium members a perk to maximize engagement bonus
- [ ] ⚠️ Avoid anything that lets paying players permanently out-grow or become un-pushable

---

### Phase 6 — Content & Variety
> Keep the game fresh past the first session.

- [ ] Chair skins & trail cosmetics (neon, metal, rainbow, themed sets)
- [ ] Multiple maps — vary size, obstacles, and hazards
- [ ] Maps with ledges and pits that synergize with the push mechanic
- [ ] Environmental hazards (conveyor belts, bumpers, bounce pads)
- [ ] Power-up orbs (rare): push shield, magnet boost, speed burst
- [ ] Special round modifiers: "Double Growth," "Low Gravity," "No Boost" — rotated

---

### Phase 7 — Social & Competitive
> Make the game sticky and shareable.

- [ ] **Global leaderboards** (DataStore-backed): biggest chair ever, most wins, weekly resets
- [ ] **In-round live leaderboard** showing current size rankings
- [ ] **Parties / friend join:** queue together, join friends' servers
- [ ] **Roblox Badges:** first win, size 50 reached, 100 pushes — free virality
- [ ] **Clip-worthy moments:** big push VFX dramatic enough to post/stream

---

### Phase 8 — Launch Readiness
> Don't get wrecked on day one.

- [ ] **Server-side size validation** — client-side scaling = exploitable; validate growth server-side ⚠️
- [ ] **Performance pass** — full server stress test (max players + max orbs + Heartbeat at 20fps)
- [ ] **Analytics** — session length, round completion rate, where players quit, conversion
- [ ] **Onboarding** — 10-second tutorial / first-time tooltips (sit, collect, press G)
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

---

## 🔢 Suggested Build Order

1. **Pick the win condition** *(recommend: biggest at the buzzer)*
2. **Phase 1 + 2** → lock the loop, add rounds → now it's a game
3. **Phase 3** → juice it → now it feels good
4. **Phase 4** → progression + DataStore → now people come back
5. **Phase 8** (anti-cheat + analytics) → pull these forward before any public release
6. **Phase 5, 6, 7** → monetization, content, social → growth and revenue

> ⚠️ Build DataStore (Phase 4) and server-side validation (Phase 8) earlier than their phase number suggests. Both are foundational and very painful to retrofit later.

---

*Maintained alongside development. Check off features as they ship.*
