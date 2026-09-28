# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Commands

```bash
uv sync                                   # install (includes browser-harness)
uv run ruff check .
uv run pytest                             # offline; mocks TypeSafe, text LLM, and browser
uv run pytest tests/test_agent.py -k stale   # single test by name
node --check jev_ultrafast/static/app.js
node --check jev_ultrafast/snapshot.js
uv build
uv run python scripts/check_guards.py     # real local Chrome, no model calls
uv run jev                                # inspector at http://127.0.0.1:8766
```

Anything run with `--env-file .env` (`examples/*.py`, `scripts/record_flights.py`, `scripts/measure_flights.py`) makes paid API calls; run it only when the user asks. Chrome connects via Browser Harness (`uv run browser-harness --doctor`).

## Architecture

One decision cycle, in `agent.py`:

1. `Browser.observe` (`browser.py`) runs `snapshot.js` in one evaluate call: visible text plus an indexed table of controls. Node identity lives in a browser-side WeakMap/Map, not CDP node IDs.
2. `model.choose` sends one TypeSafe request with an operation head and one target head per operation (`action_space` builds only compatible targets; `validate_choice` rejects anything outside them). Question wording lives in `questions.py`.
3. For `TYPE_TEXT`, `model.field_text` calls the OpenAI-compatible text helper, which must return JSON with exactly one `text` value.
4. `Browser.fresh` compares semantic fingerprints; a mismatch raises `StalePage` and the cycle re-decides. `Browser.act` re-resolves geometry and occlusion, then executes. Execution is logged before the next observation.

`demo.py` is a loopback-only HTTP server (Host/Origin/token checks) serving `static/` and driving the same `Agent`.

`docs/design.md` holds the rationale for freshness guards, wait timing, run bounds (60 actions, 120 decisions, 250 candidates), and known limits. Read it before changing guards, waits, or the text cache. `docs/performance.md` and the `docs/*measurement.json` files back the README's numbers; update them together.
