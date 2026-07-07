# Panda Puzzle Notation Guide

*The universal guideline for how puzzle moves are written in the Panda Puzzle Registry.
The canonical copy lives in the GameLoader repo (the registry hub); copies are distributed to
every app directory that might eventually pull puzzles from the centralized database. If the
format ever changes, GameLoader's copy is the truth.*

---

## 1. Where puzzles come from

All puzzles live in **GameLoader** (the hub). Any app can load them three ways:

```html
<!-- 1. The bundle: everything, one file, no build step -->
<script src="https://gameloader.pandapuzzles.com/puzzles-all.js"></script>
<script> const all = window.PANDA_PUZZLES.puzzles; </script>

<!-- 2. The Firestore client: query by type/theme with caching -->
<script src="https://gameloader.pandapuzzles.com/panda-puzzles.js"></script>
<script> const set = await PandaPuzzles.load({ type: 'mate-in-2', limit: 50 }); </script>
```
3. A **`#pz=` deep link** (see §6) for handing one puzzle to another app.

## 2. The puzzle object

```js
{
  id: 'p1f4e75394a1163',        // cyrb53 hash of the FULL FEN — see §5
  fen: '5rk1/…/6K1 w - - 0 1',  // the starting position
  side: 'w',                    // who solves: 'w' | 'b'
  kind: 'mate',                 // 'mate' (oracle-proven forced mate) | 'tactic' (best move, Stockfish-judged)
  type: 'mate-in-3',            // 'mate-in-N' for mates; 'tactic' for best-move puzzles
  mateIn: 3,                    // mates only; absent on tactics
  title: 'Reinfeld #212',
  hook: 'Can you find the forced mate in 3?',
  difficulty: 2,
  solution: [ …see §3… ],
  themes: ['back-rank-mate', 'queen-sacrifice', 'endgame', …],  // ~29 auto-derived tags
  source: 'reinfeld-1001'
}
```

## 3. How moves are notated — the solution tree

The solution is an **array of "stops"**. Each stop is one solver move plus (optionally) the
opponent's reply and any alternative defenses:

```js
solution: [
  { a: ['Qxf7+'], o: 'Rxf7', b: { 'Kh8': [ { a: ['Qxf8#'] } ] } },
  { a: ['Re8+'],  o: 'Rf8' },
  { a: ['Rxf8#'] }
]
```

| Key | Meaning |
|---|---|
| `a` | **Accepted move(s)** — the solver's move, in SAN. An array because several moves may be equally correct (e.g. `a: ['e8=Q#', 'e8=R#']`). The first entry is the mainline. |
| `o` | **Opponent's mainline reply** — one SAN string. Omitted on the final stop (the game is over) or when any legal reply loses the same way. |
| `b` | **Branches** — an object keyed by an *alternative opponent reply* (SAN); each value is a nested array of stops answering that defense. Branches nest to any depth. |

Reading the example: play **Qxf7+**. If the opponent takes (**Rxf7**), continue with **Re8+ Rf8,
Rxf8#**. If instead they play **Kh8** (a branch), it's **Qxf8#** immediately.

### The rules every producer/consumer must follow

1. **Moves are SAN** (Standard Algebraic Notation), exactly as chess.js emits them:
   piece letter + disambiguation + capture `x` + square, with `+` for check, `#` for mate,
   `=Q` for promotion, `O-O`/`O-O-O` for castling. No move numbers, no annotations (`!`, `?`).
2. **The final `a` move must deliver checkmate** (for `mate-in-N` puzzles). `mateIn` equals the
   number of stops.
3. **Solvers should compare moves loosely**: strip `+ # ! ?` before comparing a user's move to
   `a`/`b` keys, or replay both through chess.js and compare the normalized `.san`. Never
   fail a solver because of a missing `+`.
4. **Every tree must be oracle-verified** before it enters the registry
   (`GameLoader/puzzles/verify-registry.js` replays every line to mate). Don't hand-write
   trees into the database; feed them through the pipeline.

## 3.5 Authoring puzzles: Panda Markers (Lichess → puzzle)

You don't write solution trees by hand. Annotate any PGN — most naturally as **Lichess study
comments** — and GameLoader mints the puzzles:

| You type (in a move's comment) | Meaning |
|---|---|
| `[]` | This move is a puzzle answer. Prompt: "Find the best move." Answer = this one move. |
| `[#3]` | Mate-in-3 puzzle. The answer line automatically runs to the mate. |
| `[Win material]` | Custom prompt ("Win material."). Any text works. |
| `\|\|` (on a later move) | Extends the answer key + continuation through that move. |

Rules:
- The mark goes **on the answer move itself**. If the best move is a **variation** (the
  improvement over the game's blunder), mark the variation move — the puzzle position is where
  the variation branches, and the solver plays your improvement.
- Default span is the single marked move; `[#N]` runs to mate; `||` sets an explicit end.
- Multiple marks in one game/study = multiple puzzles. `[%clk]`/`[%eval]` tags are ignored.

**Two kinds, two quality gates.** Marked lines that force checkmate become `kind:'mate'`
(tagged `mate-in-N`, proven by the chess.js oracle — can't be wrong). Everything else becomes
`kind:'tactic'` ("Find the best move" / your custom prompt) and is judged by Stockfish:
`node puzzles/validate-stockfish.js` (run it whenever — it's incremental) checks that the
marked move is the engine's best AND clearly better than the second-best line (MultiPV 2,
default gap 150cp). Pass → published with the `engine-checked` theme; fail → dropped from the
registry with a report line (it stays in your local custom DB). Until judged, a tactic ships
as `engine-pending`.

The repeatable loop (three steps, every time):
1. Annotate in Lichess (comment `[]` on the answers as you analyze).
2. GameLoader → Puzzles → **Scan studies** (fetches only your new/updated studies), or paste a
   study/game URL (or raw PGN) into **Add puzzles**.
3. They're instantly playable in the Puzzles tab (`custom` + `marked` themes). Press
   **Export** to download `custom-export.json`; drop it in `GameLoader/puzzles/data/` and
   rebuild — your forced mates get oracle-validated, auto-tagged, and published to every app.

## 4. How this compares to traditional PGN

| | Traditional PGN | Panda solution tree |
|---|---|---|
| Moves | SAN | SAN — **same notation for individual moves** |
| Container | one text movetext string, move numbers, headers | JSON array of stops |
| Alternate lines | `(variations in parentheses)` | `b:` objects keyed by the opponent's deviation |
| Multiple correct answers | not expressible (one mainline) | `a: [move1, move2]` |
| Starting position | `[FEN "…"]` + `[SetUp "1"]` headers | `fen` field |
| Whose move matters | implicit from movetext | explicit `side` field |
| Machine-checkable | needs a parser (comments, NAGs, results) | direct JSON — index into stops |

**The mapping is mechanical.** This PGN:

```
[FEN "…"] 1. Qxf7+ Rxf7 (1... Kh8 2. Qxf8#) 2. Re8+ Rf8 3. Rxf8#
```
*is exactly* the tree in §3: mainline pairs become `a`/`o` stops; each PGN variation becomes a
`b` branch keyed by its first move. GameLoader's puzzle importer already converts PGN → tree
(paste PGN in the Puzzles tab, or use `+Puzzle` capture in the game viewer); marking a move
with `!` or `$1` in imported PGN sets the puzzle's starting point.

## 5. Identity — the one rule you must never break

`id = cyrb53(full FEN)` rendered as `'p' + hex`. GuessTheMove's Glicko-2 ratings
(`puzzleRatings/{id}`) and per-user stats key on it. Reference implementations:
`GameLoader/puzzles/id.js` and `guessthemove/puzzle-ratings.js`. If your app needs the id,
copy that function verbatim — **never invent a new hash or change the basis**, or every
existing rating detaches.

## 6. Handing a puzzle to another app

Deep link: `https://guessthemove.pandapuzzles.com/#pz=<base64url JSON>` where the JSON is
`{ fen, side, title, difficulty, hook, tree }` and `tree` is the §3 solution array.
Encode: `btoa(unescape(encodeURIComponent(JSON.stringify(payload))))` with `+/` → `-_`,
padding stripped. Any app can adopt the same receiver pattern (see `parsePuzzleDeepLink()` in
guessthemove/app.js).

## 7. Is it universal now?

**Yes — this is the one format.** As of July 2026 the registry unified what used to be four
incompatible shapes (GTM tuples, 99Problems tuples, Gimkit MC objects, blunder JSON). Legacy
tuple files (`reinfeld-puzzles*.js`, `puzzles-data.js`) still exist inside apps, but they carry
this same `{a,o,b}` tree in their last element, and the registry is the canonical, deduplicated,
validated superset. New apps should read `puzzles-all.js` and speak this format natively.
For the full ecosystem contract (who owns what, shipped features, roadmap), see
`PUZZLE-SHARING-HANDOVER.md` in the GameLoader and guessthemove repos.
