# Player skill badge: research and recommendation

Research note, October 2026. Goal: find the best way for Ember players to
read an opponent's rough level without turning Ember into a ladder people
grind or "play to win" on. The starting idea was the Arc System Works
colored square from Dragon Ball FighterZ, Guilty Gear Xrd Rev 2 and
BlazBlue. This note is research only; nothing here is implemented.

## Recommendation in one paragraph

Keep the ArcSys *presentation* (a small coarse color badge next to the
name, no numbers, no leaderboard, every room game counts) but replace the
ArcSys *mechanism* (a hidden same-color-only counter) with a proper
uncertainty-aware rating such as Glicko-2, computed on the bridge from
pairs of agreeing signed game reports, and shown only as one of six or
seven bands with hysteresis. The rating stays hidden; players and
spectators only ever see the band. New players show a neutral "unsettled"
badge until the rating has settled, inactive players fade to an outline
instead of dropping, and games stop counting toward promotion or demotion
when they are the tenth or so game against the same opponent in one
sitting. This gets the signal the community wants (who am I sitting down
against?) while removing every lever that makes people grind: no visible
number to climb, no rank to protect from a bad night, no mode to queue
into, nothing to win back after a break.

## What Ember has today

Relevant findings from the main repository (`Confetti3/SF4-Ember-Netplay`,
`release` branch):

- **Identity.** Every player has a locally generated Ed25519 Ember ID
  (`docs/design/identity-bridge/*`). It is free to regenerate, so it is a
  stable handle, not an anti-smurf measure. Historical records stay under
  the old ID after a key replacement.
- **Match ledger.** `server/ember/bridge/src/routes/ledger.rs` and
  `records.rs` already store matches and compute each player's record on
  read from "attempts" that both fighters accepted. The record route
  (`GET /v1/players/{ember_id}/record`) returns sets and games won and
  lost plus recent matches. A player may only read their own record.
- **Trusted results.** `docs/design/NATIVE_MATCH_RESULT.md`: the winner
  comes from the native GameManager result, published only after GGPO has
  confirmed the inputs for that frame. Disconnects are never turned into
  wins. `server/ember/protocol/src/report.rs` defines the signed
  `GameReport` each fighter sends: outcome, both Ember IDs, room and table,
  build ID, capture and confirmed frames.
- **Scope gap.** Reports exist only for bridge-run matches (tournaments
  and king-of-the-hill lobbies on a provider connection) because a report
  needs a `match_id`, `attempt_id` and `permit_id`. Ordinary public and
  private room games produce no report, so today's "online record" covers
  organized play only. Reports also do not carry which fighters were used.
- **Explicit non-goal.** The identity spec (section 1.3) says v1 "does not
  create a global ranked service". A skill badge is a smaller thing than a
  ranked service, but it is new scope and should be written down as such.
- **Rooms.** Public rooms have a VPS room host as the sole authority and
  voter; private rooms are peer to peer. Rooms play game by game on tables
  with queues, so the natural unit is a game, not a first-to-N set.

So the building blocks are there: identity, a server, a signed result
format, two-report agreement and a read-time record. What is missing is a
casual-room report path, fighter IDs in the report, the rating itself, and
the badge in the UI.

## The ArcSys colored square, as actually documented

Sources are community threads; Arc System Works has never published the
rules. Where the threads disagree, both versions are given.

| Property | What the threads say |
| --- | --- |
| Colors, low to high | DBFZ and BBTAG: light blue, dark blue, dark green, light green, yellow, orange, pink (7). Xrd Rev 2 thread: white, light blue, blue, light green, green, yellow, orange, red, pink/purple (9). |
| Start | White ("no data"). First color after "the first 30 games" (one poster) or "a few" games (others). |
| Movement | A hidden counter, described as "10/20" or "+1 win, -1 loss, get to 10". Only games against an opponent of the **same color** move it. Beating a higher color changes nothing. |
| Which games count | "All online modes, be it your friends, ranked, or just casual play" in DBFZ. One poster unsure. |
| Visibility | Nothing in game shows the counter; "you must keep track of it in your head". |
| Separate markers | A violet **name** (not square) in DBFZ marks players who "tend to leave games when losing". Xrd Rev 2 adds up/down/equals arrows showing the opponent's level **relative to you**. |
| Sentiment | Mostly confusion and arguing about the rules. One poster frames risk aversion: "if you're brave enough or don't care about rank you can go prove it". It is treated as a truer skill read than BP (the grindable points) because it cannot be farmed by volume. |

Why it suits the goal: it is coarse, has no number, has no leaderboard, is
earned in casual play rather than a dedicated mode, and is read at a glance
by both the opponent and spectators. The pink square is respected
precisely because it cannot be bought with play time.

Why it should not be copied literally:

1. **Same-color-only movement stalls in a small population.** Ember will
   often have a few dozen people online. If your badge only moves against
   your own color, many players would wait days between qualifying games,
   and a newcomer who is clearly strong would sit at green for weeks.
2. **A linear counter cannot tell a close call from a stomp.** Beating
   someone far above you counts the same as (or, in the ArcSys version,
   less than) beating a peer. This is exactly what rating systems solve.
3. **Opacity breeds folklore.** Every thread about the square is people
   contradicting each other. The badge should have a published rule, even
   if the rating behind it stays hidden.
4. **Color alone is not accessible.** Blue, green, yellow, orange and pink
   are hard to separate for color-blind players; the badge needs a second
   channel (fill level, segment count or a one-word tooltip).

## Systems compared

| System | Engine | What the player sees | Serious? | Lessons for Ember |
| --- | --- | --- | --- | --- |
| ArcSys square (DBFZ, Xrd, BB) | Hidden +1/-1 counter, same color only, all modes | 7-color square, no number | Low | The presentation to copy. Mechanism too crude. |
| Fightcade | Elo, start 1000, hidden ("?") until 10 ranked games | Letter E to S, numeric points, leaderboards, points move per game in a set | Medium | Ranked/unranked split means a chunk of players never touch ranked. Points per game inside long sets. |
| Street Fighter 6 | League Points (retention design: no loss in Rookie, win streak bonuses, no demotion below Platinum) then Master Rating, which is plain Elo | Many tiers, numbers, per-character ranks | High | LP is engineered to be grindable; MR is the honest layer. Per-character ranks let people try new fighters without fear. |
| Guilty Gear Strive, Tower (2021 to 2025) | Win/loss streaks move you between 10 floors plus Celestial; Celestial needed 5 of 6 wins monthly | A floor number; you can enter your floor or above | Medium | Pitched as "improve at your own pace" and "remove the stress of gambling for points", but players found floors mixed skill too widely and the promotion fights stressful. ArcSys replaced it with SF6-style ranks (Iron to Vanquisher, per character) in 2025 and kept the tower as a casual Duel Tower. |
| Granblue Fantasy Versus Rising | Points per win/loss, Master Points after Master, Grand Master = top 10 per character, seasonal reset | Letter grades, points, top-10 title | High | Top-tier stagnation had to be patched by raising stakes; seasons and top-N titles create exactly the "play to win" pressure Ember wants to avoid. |
| Slippi (Melee) | Opt-in ranked queue with a Bayesian rating and a displayed ordinal, seasons, leaderboard; unranked queue beside it | Bronze to Grandmaster, number, leaderboard | High in ranked, none in unranked | Closest analogue as a community netplay client. The ranked/unranked split is clean but means unranked gives you no read on your opponent at all. |

Pattern across all of these: the moment a number, a leaderboard, a season
or a reward is visible, people optimize for it. The two designs that stay
casual (the ArcSys square and GGST's old tower) both hide the number and
only show a coarse band. The tower still failed because the *promotion
event* was visible and stressful (5 of 6 wins). So the badge should also
avoid any visible promotion ritual: it simply is a different color one day.

## Rating engine choice

The badge needs an engine behind it. Options, from the search results and
the rating literature:

- **Elo.** No notion of confidence. A new player and a veteran move the
  same amount per game, so newcomers take a long time to settle and
  veterans swing on bad nights unless you tune K-factors by hand. Fightcade
  and SF6 Master Rating use it. Fine, but strictly worse than the next two
  for a small, intermittent population.
- **Glicko-2.** Rating plus a rating deviation (RD, confidence) plus a
  volatility. High RD players move fast, settled players move slowly, and
  RD grows with inactivity so a returning player re-settles quickly instead
  of being punished. Lichess runs it with a rating period of one game,
  which is the right choice for Ember's sparse, bursty play. Original paper
  recommends 10 to 15 games per period for stability, but the one-game
  variant is proven at scale.
- **Weng-Lin (OpenSkill).** Bayesian mean and sigma, usually displayed as a
  conservative ordinal (mean minus three sigma). Equivalent quality for
  1v1, permissively licensed, handles teams if Ember ever wants 2v2 tables.
- **TrueSkill.** Same family but Microsoft's license restricts it to Xbox
  Live games and non-commercial use. Avoid.

Recommendation: **Glicko-2 with a one-game rating period**, or Weng-Lin if
the implementer prefers it; the choice is not visible to players. The
bridge is Rust and the `skillratings` crate implements Elo, Glicko-2,
Weng-Lin and TrueSkill with the same interface, so the engine can be
swapped during a trial period without touching the data model. The
rating, RD and volatility are stored per Ember ID and never sent to
clients; only the band and its settled/unsettled state are.

## Proposed badge rules

These are the rules the player guide would publish. They are chosen to
give an honest read while removing every grinding incentive.

1. **Every room game counts, nothing else does.** Public and private room
   games on any table count. Training, replays and spectating do not.
   There is no ranked queue, no opt-in mode and nothing to "turn on". This
   is the ArcSys choice and the main reason the square is respected: you
   cannot avoid it and you cannot farm it in a special mode.
2. **One badge per account, not per fighter.** Per-fighter badges would be
   data-starved in a small community and push people to "main" something.
   To keep experimentation cheap, a game on a fighter with fewer than about
   20 recorded games for that player updates the rating at reduced weight
   (for example half). This needs the fighter IDs in the game report.
3. **Seven bands, no numbers, no leaderboard.** Nothing in Ember, on the
   website or in the Discord bot lists players by rating. Profiles show
   the band, the record Ember already keeps, and nothing else. Band
   thresholds are fixed rating cut-offs around the 1500 start (roughly 150
   to 200 rating points per band, calibrated once after a trial month so
   that the top band is genuinely rare), not percentiles, so a badge
   cannot change because other people played.
4. **Hysteresis.** A band changes only when the rating has crossed the
   boundary by a margin (for example 25 points) and the RD is below a
   settled threshold. One good or bad evening moves the hidden rating,
   rarely the badge. There is no promotion match, no announcement and no
   animation; the badge is simply different the next time someone looks.
5. **Unsettled, then settled.** New IDs show a neutral outline badge until
   the RD falls below the settled threshold (about 10 to 15 games in
   Glicko-2 terms, matching Fightcade's 10 and the ArcSys "first few").
   After about 60 days without a room game the badge fades to the same
   outline style in its last color rather than dropping a band. Coming
   back costs nothing but a few games to re-settle.
6. **Diminishing returns against one opponent per sitting.** Rooms play
   long sets between friends. Games beyond roughly the eighth against the
   same opponent within a few hours still count for the record but update
   the rating at a steeply reduced weight. This stops two players from
   dragging each other across a boundary in one night and removes the
   incentive to farm a friend.
7. **Only agreed, finished games count.** A game enters the rating only
   when both fighters' signed reports of the rollback-confirmed native
   result agree, as the ledger already requires for organized play. In
   public rooms the room host additionally attests that the two were seated
   at that table at that time. Disconnects, aborts and draws never move
   the rating, in keeping with the native-result design that never infers
   a win from a disconnect.
8. **A separate, quiet "leaves games" marker.** Because disconnects do not
   count, leaving when losing protects the badge. Copy DBFZ's violet name
   rather than awarding wins: if a player's share of unfinished games over
   their last N is high, their name gets a small marker. This is a conduct
   signal, kept apart from the skill badge.
9. **Accessible presentation.** Use the badge color plus a fill level or
   segment count (one of seven) so the band is readable without color, and
   a one-word tooltip or label on the profile. A thematic option that fits
   the brand: the badge is an ember whose temperature rises from ash grey
   through dull red, orange and yellow to white-hot, which reads as "how
   hot is this player" rather than "what rank are they".
10. **Where it appears.** Next to names in the room member list, on table
    cards and in queues, in the spectator view, on the profile, and
    optionally in the Discord room bot's member lists. Nowhere else.

Optional, from Xrd Rev 2: next to another player's badge, show a small
relative hint (above you, about even, below you) computed from the two
hidden ratings. It answers the actual question ("can I hang with this
person?") better than the absolute band and costs nothing extra.

## What would have to be built

In rough dependency order; sizes are impressions, not estimates.

1. **Casual game reports.** Extend `GameReport` (or add a sibling type)
   with the fighter IDs and a room-game variant that needs no `match_id`
   or `permit_id`. Helpers send one per finished room game when the player
   is signed in with an Ember ID. Public room hosts send a seating
   attestation. Medium: protocol, helper, bridge route, storage.
2. **Pairing and acceptance.** A bridge job that pairs the two fighters'
   reports for the same room, table, generation and frames, accepts the
   game when they agree, and discards or flags them when they do not.
   Small: the attempt-acceptance logic already exists for organized play.
3. **Rating state.** A table of rating, RD, volatility, last game time and
   settled flag per Ember ID, updated in the same transaction as
   acceptance, with the fighter-familiarity and same-opponent weights.
   Small with `skillratings`.
4. **Band and marker computation.** Band with hysteresis, fade state,
   leaves-games marker. Exposed through the existing player routes and
   through room state so other members' badges arrive with the member
   list. Small.
5. **UI.** The badge component in the overlay (member list, table cards,
   queues, spectate, profile), the player-guide page explaining the rules,
   and the privacy text. Medium, mostly UI work.
6. **Trial period.** Run for a month with the badge hidden, look at the
   rating distribution, set the seven cut-offs, then show it.

Open decisions for the project owner:

- Should private room games count by default, or only when both players
  have left the badge on? Counting everything is what makes the ArcSys
  square honest, but Ember has leaned toward opt-in for anything that
  leaves the player's machine.
- Should a player be able to hide their own badge? If many do, the signal
  dies; the ArcSys games do not allow it.
- Per-fighter badges later, once there is enough data, or never?
- Whether the relative hint (above/even/below) is worth the small extra
  complexity, or whether the absolute band is enough.

## Sources

ArcSys colored square:
- [DBFZ Steam: "What the Colors mean next to your name"](https://steamcommunity.com/app/678950/discussions/0/1699415798754749665)
- [DBFZ Steam: "The Colored Square and BP"](https://steamcommunity.com/app/678950/discussions/0/3377008022024577584/)
- [DBFZ Steam: "What do the squares next to Fighter Names mean?"](https://steamcommunity.com/app/678950/discussions/0/1692659769957021012/)
- [GG Xrd Rev 2 Steam: "Online icon meanings?"](https://steamcommunity.com/app/520440/discussions/0/2333276539600558620)
- [Gematsu: Xrd Rev 2 PC online lobbies detailed](https://gematsu.com/2017/04/guilty-gear-xrd-rev-2-pc-launches-june-1-online-lobbies-detailed)
- [BBCF Steam thread on lobby color squares](https://steamcommunity.com/app/586140/discussions/0/2765630416809456888)
- [BBTAG GameFAQs: "Can someone explain how your coloured square works?"](https://gamefaqs.gamespot.com/boards/213428-blazblue-cross-tag-battle/76835884)
- [UNI Exe:Late[cl-r] Steam: "Color Square online"](https://steamcommunity.com/app/801630/discussions/0/3317353727676277340/)
- [Prima Games: DBFZ online ranks and colors](https://primagames.com/tips/dragon-ball-fighterz-online-rank-titles)
- [DBFZ wiki: Danger Indicator](https://dragonballfighterz.fandom.com/wiki/Danger_Indicator)

Other fighting game systems:
- [Fightcade Patreon: How Fightcade's ranking system works](https://www.patreon.com/posts/42104017)
- [Steam discussion on SF6 League Points vs Master Rating](https://steamcommunity.com/app/1364780/discussions/0/5188758291785334402)
- [EventHubs: SF6 Master rank breakdowns](https://eventhubs.com/news/2023/dec/21/visual-breakdowns-master-ranks-sf6)
- [Automaton: Tekken 8 should adopt SF6's per-character ranks](https://automaton-media.com/en/reviews/tekken-8-should-adapt-feature-from-street-fighter-6-ranked-mode/)
- [Destructoid: Strive looking to reinvent ranked mode mentality](https://destructoid.com/?p=260720)
- [Easy Allies: Guilty Gear Strive review (floor system)](https://easyallies.com/review/guilty-gear-strive)
- [GGST Steam thread criticizing the tower](https://steamcommunity.com/app/1384160/discussions/0/3060744206325810652)
- [AV Club: Strive's upcoming ranked mode details](https://www.avclub.com/guilty-gear-strive-ranked-mode-details)
- [EventHubs: Guilty Gear Backyard updates, Duel Tower](https://eventhubs.com/news/2025/jul/25/guilty-gear-backyard-updates)
- [Guilty Gear wiki: Duel Tower](https://guiltygear.wiki.gg/wiki/Duel_Tower)
- [Automaton: Arc World Tour Finals 2024 announcements](https://automaton-media.com/en/?p=45980)
- [DashFight: GBVSR 1.52 patch (Master and Grand Master point changes)](https://dashfight.com/news/granblue-fantasy-versus-rising-1-52-patch-6404)
- [MP1st: GBVSR 1.51 adds Grand Master rank](https://mp1st.com/title-updates-and-patches/granblue-fantasy-versus-rising-update-1-51-adds-versusia-and-more)
- [Upcomer: What is Slippi Ranked](https://upcomer.com/what-is-slippi-ranked-how-to-use-the-online-super-smash-bros-melee-tool/)
- [SI: Under Night ranked guide](https://www.si.com/esports/fgc/under-night-ranked-guide)

Rating systems:
- [Wikipedia: Glicko rating system](https://en.wikipedia.org/wiki/Glicko_rating_system)
- [fooskill docs: rating system overview (Elo, Glicko, TrueSkill)](https://fooskill.readthedocs.io/en/latest/overview/rating_system.html)
- [Toph: Glicko-2 rating system](https://help.toph.co/toph/glicko-2-rating-system)
- [openskill (Weng-Lin) npm README, ordinal definition](https://www.npmjs.com/package/openskill)
- [skillratings Rust crate](https://crates.io/crates/skillratings)
- [dotgg: what makes competitive ladders feel equitable](https://dotgg.gg/understanding-ranking-systems-what-really-makes-competitive-ladders-feel-equitable/)

Ember repository files inspected: `server/README.md`,
`docs/design/identity-bridge/EMBER_IDENTITY_AND_TOURNAMENT_BRIDGE_SPEC.md`,
`docs/design/NATIVE_MATCH_RESULT.md`, `server/ember/protocol/src/report.rs`,
`server/ember/protocol/src/play.rs`, `server/ember/bridge/src/routes/ledger.rs`,
`server/ember/bridge/src/routes/records.rs`,
`server/ember/bridge/src/routes/lobbies.rs`, `docs/guides/INTEGRATIONS.md`.
