# The mini bingo widget

Every state the top-bar pill can be in, what expands when you tap it, and why each
decision was made the way it was.

Mini bingo is bingo played from inside a slot game. It appears as a 42pt circular pill
in the gameplay top bar; tapping it expands a panel below. Both halves are driven by the
same data, so this document covers them together.

For the underlying model see [`HOW_BINGO_WORKS.md`](./HOW_BINGO_WORKS.md) (plain
language) and [`BACKEND_MODEL.md`](./BACKEND_MODEL.md) (the contract).

---

## The one distinction that explains everything

**Has the player joined, or not?**

| | Pre-join | Joined |
|---|---|---|
| data available | the **room's session window** only | the **round** |
| source | `GET /rooms` (no round data at all) | the join response |
| pill shows | where the session sits in time | where the round sits in its lifecycle |
| refreshing | none needed — timestamps are fixed | comes free from the round loop |

Everything below follows from this. The lobby endpoint returns **no round information**
— that is the backend contract, not an oversight — so before joining, the pill can only
speak in terms of the session window. Round-level detail is impossible until you join.

---

## Pill states

`BingoGameplayWidget` picks one, in this order. First match wins.

### 1. Room unavailable → `BingoGameplayWidgetRoomStatus`

**When:** `useBingoRoomAvailability` returns `roomFull` or `backSoon`, i.e. the room is
at capacity, the session has ended, or the room record is missing its timestamps.

**Looks like:** greyscale full ring, a static ball in the middle, badge reading
"Room Full" or "Back Soon".

Checked **first**, deliberately: these come from the room rather than a round, and in
both cases there is no round for a phase to be derived from. Availability is read via
`useBingoRoomAvailability` rather than `useBingoRoomStatus`, because the latter reports
on *this player's join attempt* — its flags stay false when no join has been attempted,
so a spectator would see an unavailable room as available.

### 2–4. Joined: the round's phase

Chosen by `getBingoWidgetPhase(activeRound.roundStatus)`. These require a round, so they
only ever appear after joining.

| Round status | Pill | Shows |
|---|---|---|
| `TicketSalesOpen` | `BingoGameplayWidgetBuying` | countdown to the draw + "Plays in", ring draining over the sales window |
| `Countdown` | `BingoGameplayWidgetCountdown` | big seconds remaining, ring draining over the 10s countdown |
| `Calling` / `Cooldown` | `BingoGameplayWidgetCalling` | last called ball, ring showing balls-called progress, badge with either `called/total` or a to-go tier |

`Calling` and `Cooldown` share a pill because the cooldown is a result pause, not a
different activity.

### 5. Not joined → `BingoGameplayWidgetSession`

**When:** no round, so no phase. This is the fallback, and pre-join it is the *normal*
case rather than an edge one.

It resolves the session window itself, again first-match-wins:

| Condition | Pill | Label |
|---|---|---|
| `now < openTime` | countdown to `openTime` | "Sales in" |
| `openTime ≤ now < sessionStartsAt` | countdown to `sessionStartsAt` | "Plays in" |
| `sessionStartsAt ≤ now < sessionEndsAt` | full brand ring, no timer | "Now Playing" |
| otherwise | delegates to the room-status pill | "Back Soon" |

---

## Panel states

Tapping any pill expands `MiniBingoGameplayContent`, which routes on the **session**,
not on how it was opened:

```
sessionMode !== 'miniMode' and not joining  →  MiniBingoGameplayPrompt
otherwise                                   →  BingoGameplayPanelContent
```

`BingoGameplayPanelContent` then mirrors the pill's phases — a shared header (bingo
logo, title, info, close) above one of:

| Round status | Panel section |
|---|---|
| no round, more coming | spinner + "Polishing bingo balls…" |
| no round, none left | "Back Soon", no spinner |
| `TicketSalesOpen` | `BingoGameplayPanelPurchase` — buy/claim presets, prize pool, player count |
| `Countdown` | `BingoGameplayPanelCountdown` |
| `Calling` / `Cooldown` | `BingoGameplayPanelCalling` — ticket previews, called balls, winners |

So the pill and panel always agree: the pill is the glanceable form of whatever the
panel would show in full.

---

## The prompt

`MiniBingoGameplayPrompt` is the cross-sell card: bingo logo, "Mini Bingo", a dismiss X,
a two-line pitch and a **Play Now** button. It is the **only** way a player joins.

### When it opens

`useMiniBingoPrompt` opens it when **all** of these hold:

- the session is live (`sessionStartsAt ≤ now < sessionEndsAt`)
- it hasn't already been shown for this session
- the player hasn't dismissed it more than `PROMPT_DISMISSAL_CAP` (3) times, ever
- they haven't already joined

Because the widget re-renders every second, one condition covers both cases: a session
already live when the game opens prompts immediately, and a session that starts later
prompts on the tick that crosses `sessionStartsAt` — no timer needed.

### What is remembered

Persisted per account, keyed by a **session key** of `roomId:sessionStartsAt`:

| Stored | Purpose |
|---|---|
| `promptedSessionKey` | shown once per session, surviving app restarts |
| `joinedSessionKey` | re-entering a game during the same session **auto-joins** instead of asking again |
| `dismissalCount` | counts closes across all sessions; at the cap we stop asking |

### Auto-join on re-entry

Leaving a game tears the mini bingo session down, so returning would otherwise leave the
player un-joined. If the stored `joinedSessionKey` matches the current session, the
widget re-joins silently.

This is the one place a join happens without the prompt, and it is justified the same
way `appSync`'s foreground rejoin is: it restores a session **the player already opted
into**. Nobody who hasn't opted in is ever joined. The opt-in is sticky for the session
and clears when the session ends.

---

## Design decisions, and why

**Joining is deliberate, never automatic.** Mini bingo used to join on its own as soon
as a session was live — opening any slot game silently started a bingo session, opened a
game-launch session and began cycling rounds. Now the prompt's Play Now is the only
entry point. This is the change the pre-join pill states exist to support.

**The pill must never be blank pre-join.** If it rendered nothing before joining, a
player who dismissed the prompt would have no way back to it for the rest of the
session. That is why state 5 exists at all.

**Nothing pre-join refetches.** `openTime`, `sessionStartsAt` and `sessionEndsAt` are
fixed for the whole session — verified against production, where the same values came
back unchanged half an hour later. One fetch on mount plus a 1-second tick derives every
pre-join state locally. An earlier version tried to count down to the *next round* and
had to poll for fresh round data; that produced a countdown frozen at `00:00` and a
state the widget could never leave. Both bugs came from the same root cause: our mock
server invented round data on the lobby response that production never sends.

**"Now Playing", not "Play Now".** During a live session there is no round to count down
to, so the pill shows a status. The wording matches what `BingoLobbyWidget` already
shows for the same state, so the two surfaces agree. It stays tappable — it is the
manual route back to the prompt.

**Countdown format switches at 60 minutes.** `Timer`'s clock format only grows an hour
segment once there is one, so `55:00` fits the 42pt ring but `01:01:00` overflows and
gets truncated. Under an hour it counts in `MM:SS`; above, it switches to a coarse
`"2h"`, which is *narrower* and also ticks per minute instead of per second. This
matters because production's first round has a **55-minute** sales window — right at the
limit. One shared helper covers both the pre-join pill and the joined buying pill, since
both are exposed to it.

**"Back Soon" is reused for a session that is over.** It is the same greyed-out pill used
for a full room — a deliberate "nothing to play here" grouping, though it does mean
"finished for tonight" and "at capacity" look alike. Flagged in code with a `BOGI:` note
as worth revisiting.

**The panel distinguishes "still joining" from "nothing left".** A join can legitimately
return no round — a real backend state, pinned by its integration tests, not a loading
blip. `hasNextRound` separates them, so a spinner is only shown when one will actually
arrive. An unconditional spinner would hang forever at the end of a session.

**The prompt is marked as shown when it renders, not when it is requested.** The
programmatic open is refused if another widget's panel is already expanded, so marking on
request would burn the session's single prompt without the player ever seeing it.

**Any close counts as a dismissal.** The X, a swipe up, a tap outside and Esc on web do
not share an interceptable callback, so dismissal is detected by the prompt
unmounting. One known imprecision: backing out of the game with the prompt open also
counts. "Once per session" is unaffected — that is guaranteed separately — so the only
effect is the cap advancing slightly early.

**Mini mode opens with no cards.** Joining the full bingo room allocates a pool of cards
to pick from; mini mode allocates nothing, so the purchase panel starts empty and you
buy by count. That is the backend's behaviour, pinned by its tests, not a gap on our
side.

---

## Quick reference

| Situation | Pill | Panel on tap |
|---|---|---|
| Room at capacity | "Room Full", grey | prompt (not joined) |
| Session over / room closed | "Back Soon", grey | prompt (not joined) |
| Before card sales open | "Sales in MM:SS" | prompt |
| Sales open, session ahead | "Plays in MM:SS" (or `2h` beyond an hour) | prompt |
| Session live, not joined | "Now Playing" | prompt |
| Joined, sales open | "Plays in MM:SS" to the draw | buy/claim presets |
| Joined, countdown | big seconds | countdown |
| Joined, calling or cooldown | last ball + progress | tickets, called balls, winners |
| Joined, awaiting next round | previous state persists | spinner + "Polishing bingo balls…" |
| Joined, session finished | "Back Soon" | "Back Soon" |
