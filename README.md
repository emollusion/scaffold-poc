# Scaffold — PoC

A browser proof-of-concept for the ADHD executive function scaffold. Implements the
Orient → Focus → Close loop, Quick Capture (text and voice), the rule-based Focus
Engine, and Inbox Grooming. No backend, no build step, one HTML file.

## Deploy to GitHub Pages

```bash
git init
git add index.html README.md .nojekyll
git commit -m "Scaffold PoC"
git branch -M main
git remote add origin git@github.com:<you>/scaffold-poc.git
git push -u origin main
```

Then **Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**.

Live at `https://<you>.github.io/scaffold-poc/` in a minute or two.

HTTPS comes free with Pages, which matters — `getUserMedia` refuses to run outside a
secure context, so voice capture would silently fail on plain HTTP.

Nothing in the repo is over a few hundred KB. Model weights are fetched from the
Hugging Face CDN at runtime and cached by the browser, so the 100 MB per-file push
limit never comes into play.

## Speech-to-text

Open the capture sheet (**+** button), pick a model, press **Load**. First load
downloads the weights once; after that they come from the browser cache.

| Model | Download | Notes |
|---|---|---|
| `tiny.en` | ~40 MB | Fastest. Error-prone on accented or noisy input. |
| `base.en` | ~80 MB | Reasonable starting point on a phone. |
| `small.en` | ~250 MB | The size the architecture doc actually targets. Slow to load, fine once cached. |

WebGPU is used where the browser supports it, single-threaded WASM otherwise. The
status line under the picker tells you which backend you got — worth checking, since
the two differ by roughly an order of magnitude in speed.

With no model loaded, the app runs the fallback from §3.3 of the architecture doc:
audio is saved and the capture is flagged *needs review*. That path is worth trying
deliberately — it's what a failed transcription looks like in production.

### This is a deliberate deviation

The architecture doc specifies `whisper.cpp`. That needs `SharedArrayBuffer`, which
needs COOP/COEP headers, which GitHub Pages cannot send. This build runs the same
Whisper weights through `onnxruntime-web` instead, which needs no cross-origin
isolation and therefore deploys as a plain static file.

Consequence for the spike: latency numbers here compare **model sizes and device
classes against each other**. They do not predict native mobile performance, because
the inference engine is different. If you need that number, it has to come from a
real device running `whisper.cpp` natively.

Swapping the real thing in later is one function. Everything upstream only ever calls:

```js
window.ScaffoldSTT = {
  name: 'whisper.cpp-wasm',
  local: true,
  transcribe: async (blob) => "text"
};
```

If you later want genuine `whisper.cpp` in a browser, you need real headers — use
Cloudflare Pages or Netlify with a `_headers` file, or your own nginx:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

There is a service-worker workaround for Pages (`coi-serviceworker`), but it forces a
reload on first visit and `require-corp` will block the Google Fonts link in this file
unless you self-host the woff2 files.

## What this PoC answers

- Does the one-thing-at-a-time constraint survive a real phone screen
- Does the Focus Engine surface things you'd actually reach for
- Is tap-to-mark splitting low-friction enough to use (open question §10.3)
- How model size trades against latency on hardware you own

## What it does not answer

- Battery cost of on-device inference on a phone
- Real background and offline behaviour
- Flutter vs React Native packaging

## Data

Everything stays in `localStorage`. **Export all data (JSON)** on the Today screen
dumps the full state; **Reset PoC** returns to seed data. Voice clips under 1.5 MB are
inlined so they survive a reload; anything larger is session-only and says so.

## The seed data is a test fixture

Nothing in `seed()` is meant to be plausible work. Every task exists to make one
behaviour observable, and each one states in its own title what it should do — so a
wrong answer reads as wrong on the screen rather than needing to be worked out.

### Focus Engine

Pick each energy state from Orient. All four should resolve to a different task, for a
different stated reason:

| Energy | Expected task | Winning term | Margin over 2nd |
|---|---|---|---|
| High | Deep work block, 90 min | `effort` | 0.087 |
| Medium | Moderate block, 40 min | `moderate` | 0.040 |
| Low | One-minute task **A** | `small` | 0.000 (deliberate tie) |
| Scattered | Due tomorrow, 25 min | `urgency` | 0.265 |

Medium's margin is the thinnest. If you retune weights, that's the pairing that breaks
first — check it before assuming a change was harmless.

### Specific things to try

**Tie-breaking.** Low energy has two tasks scoring identically to three decimals. A
must win every time, because it's the older one. The instrumentation panel logs
`2-way tie, oldest wins` when it happens. If B ever appears, ordering has become
non-deterministic.

**The timer.** The Low winner is one minute long — the only task short enough to
actually sit through. Watch it reach zero (soft prompt, no alarm), then keep watching:
it counts up as `+MM:SS` and the ring turns green rather than stopping.

**Swap.** On High, Swap surfaces the goal-linked 50-minute task, whose winning term is
`goal`. That's the only route to seeing that reason string. Swap is offered once.

**Goal anchor.** High and Medium winners are goal-linked and show the anchor line.
The Low and Scattered winners aren't, and shouldn't show it at all.

**Orient's deadline window.** Two tasks carry deadlines: one tomorrow, one at exactly
four days — the edge of the window. Both should appear. The calendar mocks sit in the
same list and must never be selectable as Focus tasks.

**The declutter branch.** Close out both deadline-carrying tasks, then choose
Scattered. With nothing urgent left, the engine should offer ten minutes of
decluttering instead of the most-urgent rule.

**No-reason honesty.** Close out everything except *"No goal, no deadline, 90 min"*,
then choose Low. It scores zero on every term, and the engine should say so plainly
rather than manufacture a rationale.

### Inbox

Five captures, each aimed at one thing:

- **The long jumbled one** — six segments, four real items. This is the §10.3 question:
  count how many taps it takes to pull out one item. One boundary deliberately needs
  two. If it's routinely more than two, tap-to-mark is the wrong interaction.
- **The merge pair** — one thought cut in half by a pause. They're adjacent on purpose;
  *Merge with next* joins a card to the one below it.
- **The blank one** — a failed transcription, flagged *needs review*, with its audio
  marked unavailable. Lets you see the §3.3 fallback without breaking your microphone.
- **The single-line one** — splitting should be a no-op; *Make it a task* should work.

### What the fixture doesn't cover

The `age` term is weighted too low (max 0.15) to ever be the dominant reason for a
winning task, so that reason string is effectively unreachable. Either it needs more
weight or it should stop being a reason candidate — worth deciding before this becomes
the ML layer's feature set.

Once the mechanics are trusted, replace all of it with real work. The fixture proves
the engine functions; only your own tasks will tell you whether it's useful.
