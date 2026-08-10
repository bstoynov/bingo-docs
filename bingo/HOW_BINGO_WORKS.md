# How bingo works: rooms, sessions and rounds

A plain-language guide to how our bingo is organised. No technical background needed.

If you take one thing away: **a room is a venue, a session is an evening's programme at
that venue, and a round is a single game within it.**

---

## The three ideas

### Room — the venue

A room is a permanent place where bingo happens, like a physical bingo hall. It has a
name ("Crack the Code - Sun Night"), a look, and a capacity limit.

Rooms don't run bingo by themselves. They sit there, mostly closed, and open up when
there's something on.

### Session — the evening's programme

A session is a scheduled block of bingo in a room: "Sunday night, 1am until 2am".

Sessions repeat. One session can be set up to run every Sunday, or every day, and to
keep going for a set number of weeks or until a chosen date. So somebody sets it up once
and it recurs by itself.

Crucially, a session isn't one long game. It's **a running order of several games**,
each with its own start time — much like a printed programme for the evening:

> 1:00 — Game 41, cards 12¢, prizes for Mini X, Cent, Ice Cream Sundae, Coverall
> 1:06 — Game 42, cards 25¢, prizes for Lightning, Snowboard, Coverall
> 1:12 — Game 43…

### Round — one game

A round is a single game of bingo: buy cards, numbers get called, someone wins, done.
Each round comes from one entry in the session's running order, which is why the price,
the winning patterns and the prizes **change from one round to the next**.

---

## How they fit together

```
ROOM  ────────────────────────────────────────────────────────
  │   a venue: "Crack the Code - Sun Night"
  │
  └── SESSION  ────────────────────────────────────────────────
        │   this Sunday, 1:00am - 1:58am, repeats weekly
        │
        ├── ROUND  1:00am   Game 41   cards 12¢
        ├── ROUND  1:06am   Game 42   cards 25¢
        ├── ROUND  1:12am   Game 43   …
        └── …
```

One room hosts many sessions over time. One session contains many rounds. Each round is
one game.

---

## An evening, start to finish

Here's a real Sunday night in room 65.

```
 12:05am        1:00am                            1:58am          2:30am
    │             │                                 │               │
  doors        first                             last            doors
  open         game                              game            close
    │             │                                 │               │
    ├─────────────┼─────────────────────────────────┼───────────────┤
    │  buy cards  │        games run back to back   │  wind down    │
    │  55 minutes │           58 minutes            │  32 minutes   │
```

**12:05am — doors open.** The room becomes available. You can walk in and buy cards for
the first game straight away, even though it won't start for nearly an hour. Nothing is
being called yet.

**1:00am — the first game starts.** Numbers begin. From here games run one after
another, roughly every six minutes.

**1:58am — the last game finishes.** No more games tonight.

**2:30am — doors close.** The room shuts entirely.

The two "quiet" stretches are deliberate: the 55 minutes at the start let people arrive
and buy in, and the half hour at the end lets them collect winnings and wind down.

---

## What one round looks like

Take the game at 1:00am:

**Before 1:00 — cards on sale.** This is the longest part for the first game of the
night (55 minutes), because sales open the moment the doors do. For later games it's
only the few minutes since the previous game ended.

**1:00 — a short countdown.** Ten seconds. Last chance to be ready.

**1:00:10 — numbers are called.** One roughly every three and a half seconds. It's
75-ball bingo, so up to 75 numbers. Players win as their patterns complete, and there
can be several prizes in one game — a small pattern early, a full card at the end.

**When the last prize is won — a short pause.** Ten seconds to show the result.

**Then the next game.** Games are spaced six minutes apart, and a full game takes just
under five, so there's about a minute of breathing room between them.

---

## Things that surprise people

**Buying cards early doesn't mean playing early.** Sales for the first game open an hour
before it starts. Buying a card is reserving your place, not starting a game.

**The room being open doesn't mean a game is running.** From 12:05 to 1:00 the room is
open and selling cards, with no numbers being called. Same after the last game until
2:30. "Open" and "playing" are different things.

**Each game is genuinely different.** Different price, different winning patterns,
different prize amounts. The 1:00 game cost 12¢ a card; the 1:06 game cost 25¢. It's a
programme of varied games, not one game repeated.

**Prices are per card, and small.** 12¢ or 25¢ a card, and people buy many — up to 48
each. Prizes are pooled from that, which is why they run into the hundreds.

**Numbers aren't called live to your phone.** The result of a game is worked out and
sent to your phone up front; your phone then plays it out on a timer. You see numbers
appear one at a time, and that's a faithful replay rather than a live feed. It means the
game keeps working smoothly even on a patchy connection.

**Popularity varies a lot between games.** The 1:00 game had 873 players; the 1:06 game
had 33. The first game of the night gathers everyone who's been waiting.

---

## Mini bingo

Mini bingo is the same rounds, watched from inside a slot game instead of the bingo
room. Same sessions, same games, same prizes — a small bingo indicator sits at the top
of the screen while you spin.

Two differences worth knowing:

**Joining is deliberate.** You aren't put into bingo just because you opened a slot
game. A prompt offers it, and you choose. Once you've joined, bingo carries on in the
background while you play something else.

**You start with no cards.** In the full bingo room you're handed a set of cards to pick
from. In mini bingo you own nothing until you buy — the panel opens empty.

---

## Words you'll hear

| Word | What it means |
|---|---|
| **Room** | The venue. Permanent, mostly closed, opens for sessions |
| **Session** | A scheduled block of bingo in a room — an evening's programme. Repeats |
| **Round** / **Game** | One game of bingo. Many per session |
| **Cards** / **Tickets** | What you buy to play. One card is a 5×5 grid |
| **Doors open / close** | When the room accepts players — wider than the playing window |
| **Parts** | The separate prizes within one game (a line, a pattern, a full card) |
| **Coverall** | The prize for filling your whole card — usually the biggest |
| **Free cell** | The middle square, free to everyone |
| **B75** | 75-ball bingo — numbers 1 to 75 |

---

## In short

A **room** is where bingo happens. A **session** is a scheduled evening of it that
repeats week after week. A **round** is one game, and a session runs a series of them
back to back, each with its own price and prizes.

The room opens well before the first game and closes well after the last, so there are
long stretches where bingo is available but nothing is being called.

---

*For the mini bingo widget specifically — every pill state, the panel that
expands, and the prompt — see [`MINI_BINGO_WIDGET.md`](./MINI_BINGO_WIDGET.md).*

*For engineers: the technical contract — exact fields, endpoints, timings and the
scheduling model — is in [`BACKEND_MODEL.md`](./BACKEND_MODEL.md).*
