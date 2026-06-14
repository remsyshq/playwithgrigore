# Who Am I? — multiplayer party game

The card-on-the-forehead guessing game (à la the *Inglourious Basterds* tavern
scene). Each player has a secret identity that **everyone else can see but they
cannot**. Players take turns asking yes/no questions to deduce who they are.

Designed for **2+ players, co-located, on their own phones**, joining a live
room by scanning a **QR code**.

## Core insight

**The phone enforces the secret.** No paper cards, no balancing a phone on your
forehead — your identity simply *never renders on your own device*. It only
appears on everyone else's screens. When it's your turn, the others all see
"Grigore is: **NAPOLEON**" while you see "Ask a yes/no question."

Play is **co-located**: questions and answers happen **out loud**. The phones
handle word display, turn order, verdicts, and scoring.

---

## 1. Room lifecycle (outer state machine)

```
                 create
   ┌────────┐   ─────────►  ┌────────┐
   │  HOME  │                │ LOBBY  │ ◄────────── rematch
   └────────┘   ◄─────────  └────────┘
        │        join (QR)       │ host taps Start
        │                        ▼
        │                 ┌──────────────┐
        │                 │  branch on    │
        │                 │  identity mode│
        │                 └──────┬───────┘
        │            peer-assign │ │ random-deck
        │                        ▼ │
        │                 ┌────────┐│   (deck deals instantly)
        │                 │ ASSIGN ││
        │                 └───┬────┘│
        │                     │     │
        │                     ▼     ▼
        │                   ┌──────────┐
        │                   │   PLAY   │  ◄─┐
        │                   └────┬─────┘    │ turn loop
        │                        │          │ (see §3)
        │                        ▼          │
        │                   ┌──────────┐ ───┘
        └─────────────────► │  ENDED   │
                            └──────────┘
```

## 2. ASSIGN branch (the only fork — host picks per game)

```
            ┌─ mode = PEER ──────────────────────────────┐
            │  each player P  ──writes word──►  target(P) │
LOBBY ──────┤  (target = next in circle)                 ├──► all submitted? ──► PLAY
            │                                             │
            └─ mode = DECK ──────────────────────────────┘
               server deals random word to each P from chosen deck ──► PLAY
```

Net effect of both branches is identical: **every player ends up with a `word`
that only the others can see.** Everything downstream is mode-agnostic.

## 3. Turn loop (the core gameplay)

`A` = active player. The room can see A's word; A cannot.

```
        ┌──────────────────────────────────────────┐
        │   A's turn begins                          │
        │   A asks a yes/no question OUT LOUD         │
        └───────────────────┬───────────────────────┘
                            │  A taps the outcome
          ┌─────────────────┼───────────────────┐
          ▼                 ▼                   ▼
       [ YES ]           [ NO ]            [ GUESS ]
          │                 │              say it aloud
          │                 │                   │
   keep the turn      pass turn to        ┌─────┴─────┐
   (ask again)        next unfinished     ▼           ▼
          │           player          [CORRECT]   [WRONG]
          │                 │              │           │
          └────────┐        │        A finishes,   pass turn
                   ▼        │        gets a place      │
              (loop back)   └──────────┬───────────────┘
                                       ▼
                              ┌─────────────────┐
                              │  end check (§4) │
                              └─────────────────┘
```

## 4. End conditions (host toggle)

```
  win = FIRST   →  any player finishes        ──► ENDED (that player wins)
  win = RANK    →  unfinished players ≤ 1      ──► ENDED (finish order = ranking)
```

## 5. Phone-view matrix — what each screen shows at the same instant

During **PLAY**, it's A's turn:

| Role            | Phone shows                                                            |
| --------------- | --------------------------------------------------------------------- |
| **Active (A)**  | "YOUR TURN — ask a yes/no question" + `[Yes] [No] [Make a guess]`<br>(A's own word is **never** rendered on A's phone) |
| **Everyone else** | "A is: **NAPOLEON**" (big) + "answer out loud" + roster with who's finished |
| **Finished**    | "🏆 You guessed it — place #2", spectates, sees others' words          |

## 6. Per-player / room state (server-tracked)

```
player {
  id, name, connected
  word            ← secret to self, visible to all others
  finished, place ← null until they guess correctly
}
room {
  phase   : home │ lobby │ assign │ play │ ended
  mode    : peer │ deck        win : first │ rank
  turn    → index into the non-finished players
  players[]
}
```

---

## Decisions (locked for MVP)

1. **Who taps the verdict?** The **active player self-reports** — hears the
   room's spoken answer, taps Yes/No. (Alternative not used: any opponent taps.)
2. **Turn timer?** **Untimed** for MVP.
3. **Wrong guess?** **Passes the turn** (treated like a "No").
4. **Late join?** Scanners who arrive mid-game land in the lobby and join the
   **next round** (mid-round insertion is a later extension — no clean identity
   to assign on the fly).
5. **Reconnect** via a token in `localStorage` (screen-lock / refresh rejoins
   the same seat). **Host leaves** → room migrates to the next connected player.

---

## Architecture (planned)

The rest of playwithgrigore.com is static, but live rooms need shared state
synced across phones, i.e. one long-running process on the box.

```
games/whoami/          # static client (served at /whoami/ by nginx)
├── index.html         # one page; JS state machine swaps views
├── app.js             # WS connection + render per state
├── decks.js           # random-mode word lists
└── style.css          # mobile-first, matches site theme

whoami-server/         # Node + ws, in-memory rooms (no DB)
├── server.js          # rooms, turn logic, per-player sanitized state
└── package.json
```

- **Backend:** Node + WebSocket on `wp.retently.com`, run under pm2/systemd,
  listening on a local port. Room state held **in memory** (party game → no DB).
- **nginx:** one `location /whoami-ws/ { proxy_pass … }` block with WebSocket
  upgrade headers, alongside the existing game aliases.
- **Client ↔ server:** the client always connects to
  `wss://<host>/whoami-ws/`; the server broadcasts a **per-player sanitized
  state** on every change (your own `word` is stripped from your copy). Clients
  are dumb renderers of the latest state.
- **Deploy:** `deploy.sh` gains a `pm2 reload whoami` after the `git` pull.

### Protocol sketch

```
client → server : create │ join │ reconnect │ start │ submit-identity
                  │ verdict │ guess-start │ guess-result │ skip │ leave │ rematch
server → client : joined(token) │ state(sanitized room) │ error
```
