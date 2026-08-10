# Bingo backend model: rooms, sessions and rounds

How the real backend schedules bingo, what the client can see of it and when, and what
the timings actually look like in production.

Written because our mock server was originally built from this repo's Zod schemas —
which are permissive, with almost everything `.optional()` — rather than from the
backend contract. That let the mock invent data production never sends, and client
behaviour got built on top of it. This document records the contract so that can't
happen again.

For a plain-language explanation of the same model — no technical background assumed —
see [`HOW_BINGO_WORKS.md`](./HOW_BINGO_WORKS.md).
For how the mini bingo widget renders all of this, see
[`MINI_BINGO_WIDGET.md`](./MINI_BINGO_WIDGET.md).

**Sources.** `Betty-Gaming/Bingo`, `main` plus the unmerged `BETTY-9470/Mini_Bingo`
branch (mini mode is **not** in `main`), and live responses from production room 65
("Crack the Code - Sun Night"). Everything below is marked as read-from-code, observed
in prod, or inferred.

---

## 1. The three entities

### Room — `src/Bingo.Domain/Room.cs`

A durable place. Four pairs of timestamps, all nullable:

| Field | Meaning |
|---|---|
| `ShowsAt` / `HidesAt` | visibility in the lobby — **not exposed to the client** |
| `OpensAt` / `ClosesAt` | the room is open for business → `openTime` / `closeTime` |
| `SessionStartsAt` / `SessionEndsAt` | the current session's window, denormalised onto the room |
| `MaxPlayers` | capacity; drives `isAtMaxCapacity` and `MaxRoomLimitReached` |
| `Enabled`, `IsTestRoom`, `ThemeName`, `CrackTheCodeConfigurationId` | config |

```csharp
public bool IsOpen(DateTimeOffset now)
    => !(OpensAt > now) && !(ClosesAt < now);   // OpensAt <= now <= ClosesAt
```

Note `SessionStartsAt` / `SessionEndsAt` / `WidgetStateMessage` are `{ get; set; }`
while `OpenTime` / `CloseTime` are `{ get; init; }` — they are filled in
post-construction, and **only by the lobby handler**. That is why they appear in
`GET /rooms` but not in a join response.

### Session — `src/Bingo.Domain/Session.cs`

A recurring *programme* of bingo in one room.

| Field | Meaning |
|---|---|
| `StartTime`, `Duration` | when a session starts and how long it runs |
| `Frequency`, `Interval`, `Count`, `Until` | recurrence (e.g. weekly, every 1 week, until…) |
| `Schedules` | the round generators inside it |
| `RoomShowsOffset`, `RoomHidesOffset`, `RoomOpenOffset`, `RoomCloseOffset` | derive the room's windows from the session start |
| `BingoType`, `ColorHex`, `Name`, `CrackTheCodeConfigurationId` | presentation / config |

### Schedule — `src/Bingo.Domain/Schedule.cs`

One recurring round generator inside a session: `GameId` (which game config to run),
`StartTime`, `Duration`, `Frequency`, `Interval`, `Count`/`Until`.

### Round

A concrete instance produced by a schedule. Status enum, **in this order** — the
ordering matters because `IsPlaying()` is a `<` comparison:

```csharp
enum RoundStatusOptions { TicketSalesOpen, Countdown, Calling, Cooldown, Completed, Cancelled }

public bool IsPlaying() => RoundStatus < RoundStatusOptions.Completed;
```

So `IsPlaying()` is true for `TicketSalesOpen`, `Countdown`, `Calling` **and**
`Cooldown` — a round mid-draw is still "playing" and still a join candidate.

---

## 2. How a session is authored

`CreateSessionRequest` takes the session window plus
`SessionGameDto[] { GameId, StartTime, GameDisplayName }`. **Each entry becomes a
`Schedule`** — so a session is authored as a playlist of games, each with its own start
time. That is why consecutive rounds in the same session have different prices, parts
and prizes.

```
Session  "Crack the Code - Sun Night", room 65, weekly
 ├─ Schedule → Game 41 @ 01:00
 ├─ Schedule → Game 42 @ 01:06
 └─ …
```

**Inferred, not verified:** I could not find the job that materialises rounds from
schedules, and searching `RoomOpenOffset` returned no usage hits. So "the room's open
window is derived from the session offsets" is read from the field names plus the fact
that production's numbers are consistent with it — worth confirming with the backend
team before relying on it.

---

## 3. The room timeline, with real numbers

Production room 65, observed:

```
00:05        01:00                        01:58              02:30
  │            │                            │                  │
  ├─ openTime  ├─ sessionStartsAt           ├─ sessionEndsAt    ├─ closeTime
  │            │                            │                  │
  │◄─ 55 min ─►│◄────── 58 min play ──────►│◄──── 32 min ────►│
  │            │                            │                  │
  └─ cards on sale for round 1              └─ no more rounds   └─ room shuts
```

Two things this shows that are easy to get wrong:

- **The room opens long before play starts** — 55 minutes here. Cards for the first
  round are on sale that whole time.
- **The session window is a fixed absolute.** Refetching `GET /rooms` 31 minutes later
  returned an identical `sessionStartsAt`/`sessionEndsAt`, still reported unchanged
  after `sessionStartsAt` had passed. It does **not** roll forward to the next session.

---

## 4. Round lifecycle and the real intervals

`RoundTimes.Create(round)` derives the whole timeline from `startTime` and the game
config — the backend does not stream balls, it hands over the outcome and the client
replays it off a clock:

```
countdownStart = startTime
callingStart   = startTime + countdownIntervalMilliseconds
callStart(i)   = callingStart
               + i * callIntervalMilliseconds
               + (wins before i) * winPauseIntervalMilliseconds
cooldownStart  = end of last call
expectedEnd    = cooldownStart + cooldownIntervalMilliseconds
```

`TicketSalesOpen` is simply *before* `startTime`. So the sales window is not a
configured duration — it is "however long until the draw begins".

### Real production values (room 65)

| Field | Value |
|---|---|
| `countdownIntervalMilliseconds` | `10000` |
| `callIntervalMilliseconds` | `3500` |
| `cooldownIntervalMilliseconds` | `10000` |
| `winPauseIntervalMilliseconds` | **`0`** |
| `maxTicketsPerUser` | `48` |
| `maxFreeTicketsPerUser` | `10` |
| `purchasePresets` | `[1, 6, 12]` |
| `miniBingoPurchasePresets` | `[]` |
| `freeCells` | `[12]` (centre) |
| `bingoType` | `B75` — balls 1–75 |

**Worked round length.** B75 draws at most 75 balls, so a coverall round is
`10s countdown + 75 × 3.5s calling + 10s cooldown ≈ 4 min 42 s`. Observed rounds start
**6 minutes apart** (01:00, then 01:06), which leaves ~1¼ minutes of slack — the
schedule is paced so a round comfortably finishes before the next begins.

Note how far these are from the values our mock shipped with (900 ms calls, 4000 ms win
pauses, a 15-second sales window). Real bingo is much slower.

### Per-round variation, observed

| | Round 7913 | Round 7914 |
|---|---|---|
| `startTime` | 01:00 | 01:06 |
| game | Game 41 | Game 42 |
| `ticketPrice` | 0.12 | 0.25 |
| parts | Mini X, Cent, Ice Cream Sundae, Coverall | Lightning, Snowboard, Coverall |
| `userCount` | 873 | 33 |

---

## 5. What the client can see, and when

### `GET /accounts/{accountId}/rooms` — the lobby

Returns `LobbyRoomDto` = `RoomDto` + `IsAtMaxCapacity`. **Nine fields, and no round
data whatsoever:**

```
roomId  roomName  openTime  closeTime  sessionStartsAt  sessionEndsAt
widgetStateMessage  themeName  isAtMaxCapacity
+ top level: serverTime, processingDuration
```

Verified three ways: `RoomDto.cs`, `LobbyMappers.ToLobbyDto` (which maps exactly those
and nothing else), and the prod payload. The mini branch touches
`GetLobbyRequestHandler` only for occupancy refactoring.

**There is no player-facing way to learn round times before joining.** `GetRounds` can
filter by room and date but is mounted at `/management/rounds`, tagged "Rounds
Management". So any pre-join UI can only be driven by the room's session window.

### `POST /accounts/{accountId}/rooms/{roomId}` — the join

`JoinRoundResponse`:

```csharp
{ RoomDto Room; UserBalanceDto Balance; UserRoundDto? CurrentRound; bool HasNextRound; }
```

Three things worth internalising:

- **No `nextRound`** — only the `HasNextRound` boolean.
- **`CurrentRound` is nullable.**
- **The `room` here has only 5 fields** — `roomId`, `roomName`, `openTime`,
  `closeTime`, `themeName`. No session window, per §1.

The round carries the full game config, all `parts` with patterns and prizes, `result`
once drawn, `features` (e.g. `["CrackTheCode"]`), and the player's tickets. Tickets are
base64 of 25 raw bytes, positional row-major, free cells written as `0`.

### Which round you get — `JoinRoundRequestHandler`

Walks up to **10** upcoming round descriptors from `GetActiveRoundDescriptorsAsync`,
skipping any that fail `IsPlaying()` or where `now >= RoundTimes.ExpectedEnd`, then:

```csharp
ShouldForwardUserToNextRound(round, userTickets, isMiniMode):
    if (userTickets is not null)
        return round.RoundStatus != TicketSalesOpen
            && userTickets.PurchasedTicketIds.Count == 0;

    return !isMiniMode
        || round.RoundStatus != TicketSalesOpen
        || !await HasAvailableSeatsAsync(round);
```

That `&&` is the whole rule: **a round past ticket sales is skipped only if you hold no
purchased tickets in it.** Hold a stake and you stay on the round being drawn, so you
can watch your own cards. `HasNextRound` is set when a second viable round is found.

Pinned by integration tests, which are the best available spec:

- `MiniModeJoinRoom_..._TicketSalesNotOpen_AndPurchasedTickets_RoomReturnedWithCurrentRound`
  — `[TestCase]` for `Countdown`, `Calling` **and** `Cooldown`
- `..._CurrentRoundTicketSalesNotOpen_AndNoPurchasedTickets_RoomReturnedWithNextRound`
- `..._AndNoPurchasedTickets_AndNoNextRound_RoomReturnedWithNoRounds`
- `..._RoundPlayedOut_RoomReturnedWithNoRounds`

So `currentRound: null` is a real state — the tail of a session, not a loading blip.

Also note `ShouldAssignUserToCurrentRound` returns false when the request's
`CurrentRoundId` equals the candidate, which is what makes `nextRound` mean "move me
off this one".

### Mini mode vs room mode

Same handler, differing by an `IsMiniMode` flag. Routes (mini branch only):

```
POST accounts/{accountId}/mini/rooms/{roomId}
POST accounts/{accountId}/mini/rooms/{roomId}/nextRound
POST accounts/{accountId}/mini/rounds/{roundId}/tickets
POST accounts/{accountId}/mini/rounds/{roundId}/tickets/free
```

| | Room mode | Mini mode |
|---|---|---|
| tickets on join | `JoinRoundAsync` — **allocates** a pool of `Available` tickets | `GetUserTicketsAsync` — returns **only what you already hold** |
| seat | allocated | **not** allocated |
| full round | — | forwards you to the next round (`HasAvailableSeatsAsync`) |
| buy request | ticket ids | ticket **count** (`MiniModeBuyTicketsRequestPayload`) |

So a fresh mini join returns a round with **no tickets** — pinned by
`..._WithActiveRound_NoSeatAllocated` and
`..._JoinedTwice_StillNoTicketsAndNoSeatAllocated`.

### Buy vs claim: an asymmetry worth knowing

```csharp
BuyTicketsResponse      { IList<string>    TicketIds; Balance; RoundState; ServerTime; … }
ClaimFreeTicketsResponse{ IList<TicketDto> Tickets;   Balance; RoundState; ServerTime; … }
```

`BuyTicketsResponse` is shared by **both** buy routes, so a mini buy returns
**`ticketIds`, not full tickets**; only the claim endpoint returns `TicketDto[]`. Our
client's `buyMiniModeTickets` therefore always takes its `fetchBingoRound` fallback
rather than `processMiniModeTicketsResponse` — real behaviour, and an extra round trip
per purchase.

### Error codes

Join: `ValidationError`, `RoomNotFound`, `RoomNotOpen`, `MaxRoomLimitReached`,
`RoundNotFound`. Buy/claim add `TicketSalesClosed`, `NotAvailableForPurchase`,
`InsufficientFunds`, `RoundFull`, `InvalidTicket`, `GamePlayNotAllowed`,
`PlayerLimitExceeded`, `UnderMaintenance`, `AccountNotAllowed`. Errors echo
`cashBalance`/`bonusBalance`.

---

## 6. Consequences for client code

- **Nothing pre-join can show round information.** The lobby has none, so a pre-join
  widget can only derive state from `openTime` / `sessionStartsAt` / `sessionEndsAt`.
- **Those three are fixed**, so one fetch per screen is enough — no polling, and a
  countdown to them needs no refresh.
- **A countdown can be long.** 55 minutes to the first draw in production, so any
  fixed-width countdown needs a plan for the >1 hour case
  (`Timer format="clock"` only fits to `59:59`).
- **Re-joining is not neutral.** The client re-joins on app foreground (`appSync`) and
  as `joinNextBingoRound`'s error fallback; the ticket-aware rule is what keeps a
  player on a round they paid into.
- **`currentRound: null` must be handled** as a real state, distinguished from
  "still joining" by `hasNextRound`.
- **A mini buy panel opens with nothing owned.** No tickets are allocated on join.

## 7. Open questions

1. The job that materialises rounds from schedules — not located, so recurrence
   (`Frequency`/`Interval`/`Count`/`Until`) behaviour is unverified.
2. `RoomOpenOffset` and friends have no usage hits in the repo. The offsets may be
   applied elsewhere, or the room windows may be set directly.
3. Whether `GetUserTicketsAsync` returns `null` or an empty object for a player holding
   nothing. Both branches of `ShouldForwardUserToNextRound` converge on "forward" for
   that case, so it only matters on the holding-tickets path.
4. Mini mode is **unmerged** (`BETTY-9470/Mini_Bingo`), so all of §5's mini detail can
   still change before release.
