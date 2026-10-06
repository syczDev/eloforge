# API surface — what EloForge could use next

Research pass, August 2026. Nothing here is implemented; it is a menu.

Paths marked ✅ were read off the endpoint docs directly. The rest come from the
category index and should be confirmed against
[valapidocs.techchrism.me](https://valapidocs.techchrism.me/) before use.

---

## 0. What we already call

So the rest of this document is genuinely additive.

| Source | Endpoint | Powers |
|---|---|---|
| local | `/entitlements/v1/token` | auth |
| local | `/chat/v4/presences`, `/chat/v4/friends` | Friends page, game state |
| local | WebSocket `OnJsonApiEvent` | instant state changes |
| pd | `/mmr/v1/players/{puuid}` | rank, peak, act record |
| pd | `/mmr/v1/players/{puuid}/competitiveupdates` | RR change, rank chart |
| pd | `/match-history/v1/history/{puuid}` | match list |
| pd | `/match-details/v1/matches/{id}` | scoreboards, K/D sampling |
| pd | `/name-service/v2/players` (PUT) | resolving names |
| pd | `/store/v3/storefront/{puuid}` | Store page, night market |
| pd | `/store/v1/entitlements/{puuid}/{agents}` | owned agents for the picker |
| glz | `/pregame/v1/*`, `/core-game/v1/*` | live match, insta-lock |
| glz | `/parties/v1/*` | own party detection |
| valorant-api | agents, maps, competitivetiers, seasons, weapons/skinlevels, version | all names and art |

---

## 1. Riot in-game API — reads we don't use yet

Same auth we already have. These are the cheapest wins: no new plumbing, just new
calls through the existing `Remote` class.

### Player progression

| Endpoint | Gives | Feature it unlocks |
|---|---|---|
| ✅ `GET pd /account-xp/v1/players/{puuid}` | Account level, XP balance, per-match XP breakdown, **first-win-of-the-day status and next reset** | An XP/level card. "First win available" is genuinely useful and nothing else surfaces it. |
| ✅ `GET pd /contracts/v1/contracts/{puuid}` | Battlepass tier and progress, agent recruitment progress, **active daily/weekly missions with objectives and expiry** | A missions panel — "3 weeklies left, resets in 2d". Probably the single highest-value addition here. |
| `GET pd /personalization/v2/players/{puuid}/playerloadout` ✅ | Equipped skins, sprays, player card, title, **incognito flag** | Show your own loadout; explain *why* a name is hidden. Own PUUID only. |
| `GET pd /restrictions/v3/penalties` | Active queue restrictions / bans | Warn before queueing. |

### Ranked and comparison

| Endpoint | Gives | Feature it unlocks |
|---|---|---|
| ✅ `GET pd /mmr/v1/leaderboards/affinity/{region}/queue/competitive/season/{actId}?startIndex=&size=&query=` | Full ranked ladder: name, rank, RR, wins, leaderboard position. Supports name search. | A Leaderboard page. Also lets you answer "what number am I" for Immortal+, and show where a lobby sits against the ladder. |
| `GET pd /store/v1/wallet/{puuid}` ✅ | VP / Radianite / Kingdom Credits balance | Put a balance on the Store page — it currently shows prices with nothing to compare against. Currency UUIDs come from `valorant-api.com/v1/currencies`. |

### Inventory

| Endpoint | Gives | Feature it unlocks |
|---|---|---|
| ✅ `GET pd /store/v1/entitlements/{puuid}/{ItemTypeID}` | We only ask for agents. The same endpoint serves **skins, skin variants, sprays, buddies, cards, titles, contracts** | A collection/inventory page, and "you already own this" on the store. |

Item type IDs:

```
agents         01bb38e1-da47-4e6a-9b3d-945fe4655707   (already used)
contracts      f85cb6f7-33e5-4dc8-b609-ec7212301948
sprays         d5f120f8-ff8c-4aac-92ea-f2b5acbe9475
gun buddies    dd3bf334-87f3-40bd-b043-682a57a8dc3a
cards          3f296c07-64c3-494c-923b-fe692a4fa1bd
skins          e7c63390-eda7-46e0-bb7a-a6abdacd2433
skin variants  3ad1b2b2-acdb-4524-852f-954a76ddae0a
titles         de7caa6b-adf7-4588-bbd1-143831e786c6
```

### Cheap fix to something we already do

✅ `match-history` takes `startIndex`, `endIndex` and `queue`. We pass a count and
filter queues client-side. Using `queue` server-side would make the history page's
filter chips fetch less, and `startIndex` would let it page back further than the
current window instead of relying entirely on the archive.

---

## 2. Riot in-game API — writes

These exist and are documented. **All of them are automation of the game client**,
i.e. the same category as insta-lock, only more so. Listing for completeness, not
recommending.

- Party: invite, kick, set ready, change queue, generate/join by code, set accessibility
- Matchmaking: enter queue, leave queue
- Custom games: start, configure
- Pregame: select / lock / **quit**
- Current game: **quit** (i.e. dodge)
- Chat: send message

Read endpoints are hard to distinguish from the game's own traffic. Writes are not.
Queue-dodging and matchmaking automation are the things Riot actually acts on.

---

## 3. Riot local API — more from the client we're already connected to

Free: we already hold the lockfile credentials and a WebSocket.

| Endpoint | Gives | Feature |
|---|---|---|
| `GET /chat/v4/friendrequests` | Pending incoming/outgoing requests | Friends page completeness |
| `GET /chat/v6/conversations` + `/chat/v6/messages` | Party / pregame / in-game chat rooms and history | Read team chat in the app. Reading is passive; sending is not. |
| `GET /riotclient/region-locale` | Region and locale straight from the client | **Would replace the 42 MB log scan in `region_from_log()`.** Worth checking first — it may make that whole function unnecessary. |
| `GET /rso-auth/v1/authorization/userinfo` | Account info incl. email-verified, region | Settings/About |
| `GET /product-session/v1/external-sessions` | Which Riot games are running | Detect VALORANT launching before presence updates |
| `GET /help` (local) | Live list of every endpoint this client build exposes | Discovery — this is how you find new ones without waiting for docs |

---

## 4. valorant-api.com — content we don't pull

No auth, cached to disk already, cheap to add.

| Endpoint | Gives | Feature |
|---|---|---|
| `/v1/playercards` | Card art | Show players' equipped cards on the scoreboard — big visual upgrade |
| `/v1/playertitles` | Title text | Same |
| `/v1/currencies` | VP / RP / KC names + icons + **UUIDs** | Needed for the wallet above |
| `/v1/gamemodes` | Mode names, icons | Better queue labels than our hardcoded `friendly_queue()` map |
| `/v1/weapons` | Full weapon data, stats, skins tree | Weapon-specific stats; store detail |
| `/v1/sprays`, `/v1/buddies` | Art | Inventory page |
| `/v1/ceremonies` | Ace/clutch/flawless ceremony ids | Label those moments in match detail |
| `/v1/competitivetiers` | Already used | — |
| `/v1/version` | Already used | — |

`/v1/gamemodes` is worth doing regardless — our queue-name mapping is a hardcoded
list in two files that silently falls through to the raw id for anything new.

---

## 5. Riot's *official* API

[developer.riotgames.com](https://developer.riotgames.com/docs/valorant)

| API | Gives |
|---|---|
| `val-content-v1` | Content ids and localised text |
| `val-match-v1` | `/matches/{id}`, `/matchlists/by-puuid/{puuid}`, `/recent-matches/by-queue/{queue}` |
| `val-ranked-v1` | `/leaderboards/by-act/{actId}` |
| `val-status-v1` | `/platform-data` — server status and incidents |

**The catch, and it's a big one.** Riot requires every VALORANT app to use Riot Sign
On and get user opt-in, which needs a **production key**. Personal keys are
explicitly unsupported for this, and `val-match-v1` needs production too. Production
approval is a review process, and Riot has historically been reluctant about
third-party VALORANT tools.

The one piece worth having regardless: **`val-status-v1` is public** and would let
you show "NA servers are having issues" instead of the app looking broken when Riot
is down.

---

## 6. Third party

### HenrikDev unofficial API — [docs.henrikdev.xyz](https://docs.henrikdev.xyz/)

Wraps the in-game API server-side, so it works **without the Riot client running**.
Needs a free API key; rate limited; paid tier for higher limits.

Endpoints: account, MMR (v1–v3), MMR history, matches (v3/v4, paginated), match
details, stored matches, leaderboard (v1–v3), **Premier** (team search, leaderboard,
team history), esports schedule, store offers, crosshair image generation, queue
status, version.

Two things it gives that we cannot get ourselves:

- **Premier** — team rosters, standings, match history. Nothing in the in-game API we use covers this.
- **Lookup by name#tag without the client open.** Today EloForge can only ever see the signed-in account. This would let you look up a friend's profile from a search box.

Trade-off: your users' queries go through someone else's server, and it becomes a
dependency you don't control. Worth it for Premier and search; not worth it for
anything you can already get locally.

### VLR / esports

HenrikDev also proxies VLR.gg: events, matches, teams, players. Pro match schedule
and results. Nice-to-have panel; unrelated to tracking your own play.

---

## Shortlist

If it were me, in order:

1. **Contracts** — missions and battlepass. Most-used thing we don't show.
2. **Account XP** — first-win-of-the-day timer.
3. **Leaderboard** — new page, and ladder context for Immortal+ lobbies.
4. **Wallet + `/v1/currencies`** — completes the Store page, ~30 lines.
5. **`region-locale`** — may delete the log scan entirely.
6. **`val-status-v1`** — public, no key, stops outages looking like our bug.
7. **Player cards/titles** — cosmetic, but makes the scoreboard look like the game.

1, 2, 4 and 5 are all "one call, existing auth, existing patterns".

---

## Sources

- [Valorant API Docs (techchrism)](https://valapidocs.techchrism.me/) — in-game and local endpoints
- [valorant-api.com](https://valorant-api.com/) — content and assets
- [Riot Developer Portal — VALORANT](https://developer.riotgames.com/docs/valorant) — official APIs
- [HenrikDev API docs](https://docs.henrikdev.xyz/) — third-party wrapper
- [valorant-api-docs on GitHub](https://github.com/techchrism/valorant-api-docs) — source for the endpoint index
