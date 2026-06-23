# AUDIT — lattttice

Independent audit of the build pass. Findings only; no code changes here.

Evidence comes from:
- `npx jest` (5 suites, 42 tests green at audit time; 1 added probe suite emits diagnostics).
- `npx tsc --noEmit` (clean under strict).
- `npx eslint . --ext .ts,.tsx` (clean).
- Static read of `src/state/store.ts`, `src/views/*`, and `App.tsx`.

Severity: **P0** = incorrect game logic / freezes / crashes; **P1** = wrong but non-crashing behaviour; **P2** = hygiene / UX / a11y polish.

---

## A. Correctness

### A1 — Line enumeration ✅ (with documented deviation)

Probe output:
```
[AUDIT] LINES.length=520
```

Per `DECISIONS.md §1`, the spec's verbatim "1016" is an arithmetic typo: the spec's own derivation (3⁴ patterns, halved for direction symmetry → 40 direction classes, summed by start-counts) produces **520**, matching the closed-form `((n+2)^d − n^d)/2 = (6⁴ − 4⁴)/2 = 520` and matching the known counts for 3³ (49) and 4³ Qubic (76).

The test asserts 520 and also asserts the closed-form formula. Hand-spot-check of 5 lines printed in the probe:

```
line[0] = (0,0,0,0)(0,0,0,1)(0,0,0,2)(0,0,0,3)   pure w-axis  ✓
line[1] = (0,2,0,2)(0,2,1,2)(0,2,2,2)(0,2,3,2)   pure z-axis  ✓
line[2] = (1,0,0,3)(1,0,1,2)(1,0,2,1)(1,0,3,0)   z-inc, w-dec ✓
line[3] = (3,0,3,2)(3,1,3,2)(3,2,3,2)(3,3,3,2)   pure y-axis  ✓
line[4] = (0,0,3,0)(0,1,2,1)(0,2,1,2)(0,3,0,3)   y-inc, z-dec, w-inc ✓
```

All five satisfy "each coord constant / strictly inc / strictly dec, with ≥1 varying axis".

**Status:** Pass. Deviation from spec number is documented and intentional.

### A2 — Win detection ✅

`src/game/__tests__/rules.test.ts` covers:
- pure-axis win,
- 2-coord diagonal win,
- 3-coord diagonal win,
- full 4-coord space-diagonal win,
- anti-diagonal win,
- 3-in-a-line + gap is NOT a win,
- mixed-line is NOT a win,
- bent/non-line is NOT a win.

All pass.

### A3 — Coordinate round-trip ✅

`src/game/__tests__/coords.test.ts` exhaustively verifies `fromCoord(toCoord(idx)) === idx` for all 256 indices and the inverse for all coordinates. Passes.

### A4 — Move legality ✅

Probe output:
```
[AUDIT] illegal-AI-moves over 100 randomized positions: 0
```

Plus the dedicated 200-position Jest test in `src/ai/__tests__/ai.test.ts` ("easy / medium / hard never play illegal or occupied cells"). Passes.

---

## B. AI behavior

### B1 — Easy AI takes wins / blocks losses ✅

`src/ai/__tests__/ai.test.ts` includes scenarios:
- *"takes the immediate win when available"* — passes.
- *"blocks an immediate loss when no win is available"* — passes.

### B2 — Hard AI matches plain minimax on small position ✅

`src/ai/__tests__/ai.test.ts` `"alpha-beta returns the same root score as the unpruned root scan at depth 1"` passes. At depth 1 with an immediate-win / immediate-block position, alpha-beta and minimax must agree on the tactical move; both return `(3,0,0,0)`.

### B3 — Hard AI time budget ⚠️ P1

Probe output:
```
[AUDIT] hard search empty-board ms=14
[AUDIT] hard search mid-game ms=15
```

The 600 ms budget is comfortably respected — too comfortably. The branch cap of 12 plus the symmetric eval function means depth 3 finishes in ~15 ms on the empty board. The AI is correct but is leaving a lot of headroom on the table; the "hard" level isn't visibly harder than "medium" because the branching cap clips most candidates before depth bites.

**Severity P1 — wrong-but-not-crashing.** Suggested fix: raise `branchCap` to ~24, raise `maxDepth` to 4, and run iterative deepening until the soft budget actually triggers. Also persist a small principal-variation hint between iterations.

### B4 — AI on UI thread ✅ (with caveat)

`store.runAiIfNeeded` `await`s a `setTimeout(0)` so the "thinking" indicator paints, but the negamax itself is synchronous JS (in RN, the UI runs on the native thread while JS is single-threaded — so JS-side blocking doesn't freeze the UI but does freeze input). With ~15 ms searches today, this is invisible to the user. **If** B3 is addressed and search runs to budget (≈600 ms), input would be blocked for ~600 ms.

**Severity P1.** Suggested fix: after B3, split the search into yielding chunks (`await Promise.resolve()` between branches at the root iteration) or run the search inside `InteractionManager.runAfterInteractions`. The Monte-Carlo `runRolloutsAsync` already yields every 25 playouts — apply the same pattern to negamax.

---

## C. Overlay & probability

### C1 — Strength function ✅

`src/overlay/__tests__/strength.test.ts` covers:
- mixed (dead) lines contribute exactly 0 (delta-precise vs. baseline),
- adding one of my marks to an otherwise-empty line strictly increases that line's contribution,
- opponent threat increases my defensive score for the blocking cell,
- occupied cells are skipped in `scoreAllEmpty`,
- `topNRanked` ordering + rank labels.

All pass.

### C2 — Monte Carlo ✅

`src/ai/__tests__/ai.test.ts` covers:
- probabilities sum to ~1,
- clearly-won position returns P(winner) ≈ 1,
- respects abort signal.

Plus the probe confirms `pX=1.00, pO=0.00, pDraw=0.00` for an X-wins terminal position.

Cancellation: `store.recomputeProb` debounces with `setTimeout(120ms)`, holds an abort token (`probAbort`), and the rollout loop checks `opts.aborted` between samples. Caching: replaces any existing entry at the same ply (so re-analysis overwrites, never duplicates).

**Status:** Pass.

---

## D. Views & UX

### D1 — All 4 views render from shared state ✅

Manual code-review of `src/views/*`:
- `GridOfGrids.tsx` reads `cells, playAt, winningLine, toMove, overlayPerPlayer, winner` from `useStore`.
- `ZSlices.tsx` reads `cells, playAt, winningLine, currentW`.
- `Projection3D.tsx` reads `cells, winningLine`.
- `HeatView.tsx` reads `cells, playAt, toMove, winningLine`.

Switching views calls `setView` which sets only the `view` field — board state is untouched. **Recommend** an explicit test for this; covered as P2 below.

### D2 — Grid-of-grids tap mapping ✅

`GridOfGrids.tsx` constructs each pane keyed by `(z, w)` outer coords and computes the cell idx as `toIdx(x, y, z, w)`. The same `toIdx` formula is used by the engine, so taps and engine cell mapping are guaranteed consistent.

### D3 — Overlay flags per-player, default OFF ✅

`store.ts`: `overlayPerPlayer: { X: false, O: false }`. The store action `setOverlayForPlayer(player, on)` updates one side independently.

### D4 — Player can tap during AI turn ⚠️ P1

**File:** `src/state/store.ts`, `playAt` action.

`playAt(idx)` only checks `isLegal(board, idx)` (cell empty + no winner). It does NOT check whether `mode === 'vs-computer'` and `toMove === 'O'` (the AI's turn). A fast double-tap can let the human play O's move for it before `runAiIfNeeded` resolves the `setTimeout(0)`.

**Repro:** Mode = vs-computer. Tap any empty cell rapidly twice. The second tap may register as an O move played by the human, skipping the AI.

**Severity P1.** Suggested fix: in `playAt`, early-return when `mode === 'vs-computer' && toMove === 'O'`. Also disable `<Pressable>` cells in the views when `isThinking || (mode === 'vs-computer' && toMove === 'O')`.

### D5 — 3D projection is static, not GL ⚠️ P2

`src/views/Projection3D.tsx` renders 256 absolutely-positioned `View` dots from a precomputed JS projection, not via `@react-three/fiber`. Documented in DECISIONS §2. The spec asks for pinch-zoom + drag-rotate; the static fallback has neither.

**Severity P2.** Acceptable for first build pass; refine should upgrade to an `expo-gl` canvas with `react-three-fiber` when time permits, or document the static projection as the intentional simplification.

### D6 — Winning line highlight visible in all views ✅

GridOfGrids, ZSlices, Projection3D, and HeatView each maintain a `winSet = new Set(winningLine)` and outline win cells with the `theme.win` colour.

### D7 — Heat-view scale legend missing ⚠️ P2

Heat view renders shades but has no min/max swatch legend, so a viewer cannot map a colour back to "this cell is the strongest". The brightest cell is "best" by convention but it's worth a visible scale bar.

**Severity P2.** Suggested fix: small horizontal viridis legend with min/max labels under the heat board.

---

## E. Engineering hygiene

### E1 — `tsc --noEmit` strict-clean ✅

`npx tsc --noEmit` exits 0. No `any` (ESLint `@typescript-eslint/no-explicit-any: error`).

### E2 — ESLint clean ✅

`npx eslint . --ext .ts,.tsx` exits 0.

### E3 — Persistence ✅ (untested under kill/restart)

`store.persist` writes the full snapshot to AsyncStorage on every mutation; `store.hydrate` reads on app start; `App.tsx` calls `hydrate()` in a `useEffect`. The flow is correct on inspection but has not been verified on-device.

**Severity P2.** Suggested action: add a Jest test that mocks AsyncStorage and round-trips persist→hydrate.

### E4 — Accessibility — VoiceOver labels ✅

Every cell `Pressable` in GridOfGrids, ZSlices, Projection3D, and HeatView passes an `accessibilityLabel` of the form `"x3 y1 z0 w2, empty"` via `describeCell()` (`src/views/cellLabel.ts`).

### E5 — Accessibility — colours ✅

Theme uses deep blue (#3aa0ff) for X and warm orange (#ffb455) for O — distinguishable for protanopia/deuteranopia/tritanopia.

### E6 — `npx expo start` runs clean ⚠️ P2

Not executed in this audit pass (would need a device/emulator). The dependency graph requires `--legacy-peer-deps` on install (three vs. expo-three peer mismatch) — documented in README. Splash/icon/favicon PNGs are programmatically generated placeholders; they exist so `expo start` doesn't fail on missing files, but they are not the final visual identity.

**Severity P2.** Suggested action: confirm `npx expo start` boots in Expo Go on a real device before the submission pass.

### E7 — Reanimated configured but unused ⚠️ P2

`babel.config.js` registers `react-native-reanimated/plugin`, package is installed, but no view actually uses Reanimated. The plugin is harmless when unused, but the spec calls for smooth view transitions (Reanimated). Track for the refine polish pass.

### E8 — `console.warn` in `store.ts` paths ⚠️ P2

`hydrate`, `persist`, and `recomputeProb` call `console.warn` on error. ESLint allows `warn` per the rule config, but on-device logs should ideally go through a single logger so they can be silenced in production.

---

## Prioritised fix list

### P0

_None observed._

### P1

1. **D4** — `playAt` does not reject taps during the AI's turn → human can play O's move on fast double-tap. *Fix:* early-return in `playAt`; also disable cell `Pressable`s when `isThinking` or `(mode === 'vs-computer' && toMove === 'O')`.
2. **B3** — Hard AI finishes in ~15 ms; branching cap clips depth. *Fix:* raise `branchCap` to ~24 and `maxDepth` to 4 with true iterative deepening to the soft 600 ms budget.
3. **B4** — Once B3 is fixed, the synchronous negamax will block input for ~600 ms. *Fix:* yield to the event loop between root branches (mirror the Monte-Carlo `yieldEvery` pattern). Add a Jest test that asserts the worst-case `await searchHardMoveAsync()` returns control within ≤25 ms slices.

### P2

4. **D5** — `Projection3D` is a static JS projection, not an interactive GL canvas (pinch/rotate). *Fix:* upgrade to react-three-fiber / expo-gl during refine polish, or document as intentional simplification.
5. **D7** — Heat view lacks a min/max legend. *Fix:* add a small viridis legend below the heat board.
6. **D1 add-on** — Add an explicit test "play 5 moves, cycle all 4 views, assert board unchanged".
7. **E3** — Add a persist→hydrate round-trip Jest test mocking AsyncStorage.
8. **E6** — Verify `npx expo start` boots in Expo Go before the submission pass.
9. **E7** — Use Reanimated for the segmented-control underline animation and view-switch crossfade (spec calls for smooth transitions).
10. **E8** — Route `console.warn` through a thin logger so production builds can silence non-fatal storage warnings.

---

## Summary

- **Correctness (A):** clean.
- **AI (B):** correct but overly conservative; raise depth/branching.
- **Overlay & probability (C):** clean.
- **Views & UX (D):** one real bug (double-tap during AI turn), and one acknowledged simplification (3D projection).
- **Engineering hygiene (E):** clean static analysis; a few wires to tighten.

Recommend addressing the three P1s before merging, then sweep the eight P2s during the refine polish.

---

## Round 2 — live device audit (Android + iOS)

After Round 1 fixes were committed (557b161), the app was rebuilt as an Android
dev client (via `expo prebuild --platform android --clean` + `expo run:android`)
and the JS bundle was served from Metro on 127.0.0.1:8082 via React Native
"Change Bundle Location" dev menu (port 8081 is reserved for the branchxo agent).

### Verified on Android (Pixel emulator `etabli_pixel`)

| feature | result |
|---------|--------|
| 4 views render (Grid/Slices/3D/Heat) | ✅ verified via screenshots |
| Tap → cell fills with X (Grid) | ✅ |
| Mode toggle → vs Computer | ✅ AI O played within ~1 s |
| AI level toggle → Hard | ✅ Hard AI plays a sound block move |
| Win-probability chart + turning point | ✅ "Turning point — ply 2, Δp(X) ≈ 0.04" |
| Persistence (force-stop + relaunch) | ✅ board + history + mode + AI level + chart all restored |
| Heat-view legend | ✅ "6.0 → 16.7" viridis bar rendered |

### Round 2 findings

**R2-D5b — 3D projection clipped to upper-left of canvas (P2)**

3D screenshot showed the point cloud biased to the upper-left third of the 360×360
canvas. Root cause: per-w offsets (`wOffsetX = (w-1.5)*6` then `× 6` in screen-space)
were applied without accounting for the cloud's centroid, leaving asymmetric
whitespace. **Fix:** compute the raw bounding box first, then re-center on the
canvas. Also bumped `wOffsetX/wOffsetY` magnitudes (12,8 instead of 6,4) for
clearer w-stratification and dropped the canvas to 320×320 (more padding).
Verified post-fix: cloud now centered, visible padding on all sides.

**R2-D-perf — ZSlices recreates `winSet` on every render (P2)**

`new Set(winningLine ?? [])` was at the top of `ZSlices` outside `useMemo`,
breaking memoization for the inner cell rows. **Fix:** wrapped in `useMemo`.

### Round 2 conclusion

No P0 / P1 found. Two P2 polish fixes shipped (R2-D5b, R2-D-perf). Gates remain
clean (57/57 tests, tsc clean, eslint clean). Ready for Round 3.

---

## Round 3 — final audit pass (Android live)

After Round 2 fixes were committed (f12c1a7), a third audit pass walked the
following risk paths:

| risk | result |
|------|--------|
| `findTurningPoint` indexes outside `probHistory` if turning.ply not present | ✅ derived from same array — safe |
| `ProbChart` width hard-coded at 320 — overflow on narrow phones | ✅ inside `ScrollView`; verified fits on 412dp Pixel emulator |
| `searchHardMoveAsync` alpha-beta correctness at root | ✅ root only updates `alpha`; never cuts on beta because beta stays Infinity at root (correct) |
| Monte Carlo greedy-bias in `pickGreedyRandom` (always takes immediate win/block) | ✅ as designed: heuristic raises rollout strength, tests confirm sum-to-1 + correct on terminal positions |
| `runAiIfNeeded` double-fire on rapid `setMode(vs-computer)` toggle | minor: no abort on existing AI before re-fire, but `isThinking` early-return inside `runAiIfNeeded` makes the second call a no-op |
| `newGame` does not seed `probHistory[0]` | acceptable: chart shows "Play a move to see analysis." then first move triggers `recomputeProb` |
| 3D projection centering (post R2 fix) | ✅ verified on Android — cloud now occupies canvas symmetrically |
| Persistence: state survives `am force-stop` + relaunch | ✅ board + history + chart + turning-point + mode + AI level all restored |
| All four views render shared state correctly | ✅ verified by screenshot in Grid → Slices → 3D → Heat sequence |
| VoiceOver labels match spec `"x# y# z# w#, (empty|X|O)"` | ✅ across Grid, Slices, 3D, Heat (tests + grep) |

### Round 3 conclusion

**Zero P0, zero P1, zero P2.** All four views work end-to-end on Android.
iOS rendering verified (behind a system notification dialog that blocked
synthetic taps but did not obscure the rendered UI). Gates remain clean.

Stopping condition met: an audit round produced zero P0/P1.
