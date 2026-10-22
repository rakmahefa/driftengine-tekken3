# Session Handoff — DriftEngine to Bevy Migration

**Last updated:** 2026-10-22  
**Repository:** https://github.com/rakmahefa/driftengine-tekken3  
**Current documentation branch:** `docs/bevy-migration-audit`  
**Pull request:** https://github.com/rakmahefa/driftengine-tekken3/pull/1  
**Status:** Documentation audit created; PR is open and has not been merged. No runtime migration has started.

## 1. User's objective

Replace DriftEngine with a native Rust application using the Bevy engine for the Tekken 3 remake. Linux is the first priority because the user runs Linux.

Immediate scope was intentionally limited to **analysis and an audit document only**. The later implementation should start with a minimal native Linux application that runs **without a ROM or disc data**, then allow the user to test an AppImage locally with their own legally obtained game files.

Do not start implementing the engine migration unless the user resumes the project and asks to proceed. This file exists to restore context in a later session.

## 2. Repository and current GitHub state

- Repository: `rakmahefa/driftengine-tekken3` (fork of `Emulator000/driftengine-tekken3`).
- Default branch: `main`.
- Base commit reviewed for the audit: `df72437dc70d28cef51f32aff63d8e19a4cc7c20`.
- Documentation branch: `docs/bevy-migration-audit`.
- PR #1: [docs: audit migration from DriftEngine to Bevy](https://github.com/rakmahefa/driftengine-tekken3/pull/1).
- The branch initially added `audit.md`; this handoff file is being added to the same branch/PR so the audit and restart notes stay together.
- The PR has not been merged. Verify its state and branch contents before resuming, as GitHub state may have changed while the user was away.

## 3. Documents

- [`audit.md` on the documentation branch](https://github.com/rakmahefa/driftengine-tekken3/blob/docs/bevy-migration-audit/audit.md) — repository inventory, migration assessment, work estimates by subsystem, major risks, proposed milestones, and Linux/AppImage requirements.
- [`SESSION_HANDOFF.md` on the documentation branch](https://github.com/rakmahefa/driftengine-tekken3/blob/docs/bevy-migration-audit/SESSION_HANDOFF.md) — restart context and next steps.

The audit estimates a full-feature native migration at approximately **178–325 engineering person-days** (about 9–16 full-time engineer-months). This is an early planning range, not a guarantee. The biggest uncertainties are renderer parity, preserving deterministic simulation behaviour, the custom audio pipeline, and native online peer connectivity.

## 4. Findings to remember

The current repository is a TypeScript/browser application with DriftEngine integration, not a Rust application.

Useful high-level architecture:

- `src/sim/`: game simulation and state progression; intentionally independent from the engine and browser. This is a major architectural advantage, but the TypeScript cannot be used directly as native Rust by Bevy.
- `src/present/`: custom 2D/3D presentation, stages, fighter models/skinning, effects, HUD, menus/scenes, and audio playback; much of this requires a genuine port away from DriftEngine APIs.
- `src/app/`: browser window/lifecycle, fixed 60 Hz loop, controls, saves/settings and diagnostics; replace with native Bevy/Rust platform services.
- `src/assets/` and `scripts/`: typed runtime asset validation plus Node.js disc extraction/import/conversion pipeline.
- `src/net/` and `server/`: online session logic, DriftEngine transport/browser peer facilities, and Node relay.
- `tests/`: existing regression tests and architecture boundary checks can help preserve behaviour during the migration.

Recommended architecture from the audit:

1. Keep Node.js disc/extraction/importer scripts initially; do not rewrite both importer and runtime contract at once.
2. Document/freeze the generated asset format, then implement Rust loaders and validation.
3. Port the simulation to Rust incrementally and preserve the explicit frame-by-frame sequencing and integer/fixed-point semantics where required for determinism.
4. Build a Bevy renderer with a vertical slice (one synthetic/test fighter, one test stage, camera, animation and minimal HUD) before porting the entire renderer.
5. Get offline play working before restoring online transport.
6. Introduce Linux AppImage packaging only after the no-ROM native startup path is stable.

## 5. Next actions when the project resumes

Follow this sequence rather than trying to port the complete project in one step.

### Step 1 — Re-establish repository state

- Inspect the current state of PR #1 and whether it has been merged.
- If still open, review and merge it when the user is satisfied with the docs.
- Update local/fork state from `main` and make a new implementation branch from the latest `main`, keeping the docs history.
- Re-read `AGENTS.md`, `audit.md`, `docs/ENGINE.md`, `package.json`, and `tests/architecture.test.mjs` before implementation.

### Step 2 — Linux-first Bevy proof of concept

- Create a Rust/Cargo application with a pinned Bevy release and a recorded Rust toolchain.
- Add startup, diagnostics and a fixed-step 60 Hz game clock without importing or linking DriftEngine.
- Make the app open and render a simple test scene using synthetic assets only.
- Add a clear **no game data installed** state. Missing user data must not be treated as an unexplained startup crash.
- Add a `--smoke-test`/equivalent mode that verifies startup and the no-data state, emits an unambiguous success/failure result, then exits.

**Exit gate:** reproducible Linux debug/release builds, Rust tests, and a passing startup smoke test without ROM/disc files.

### Step 3 — Asset and simulation vertical slice

- Define the Rust-side asset contract using synthetic fixtures first.
- Keep the current Node.js importer as the source of generated assets for now.
- Port deterministic math/input and then a minimal offline match into Rust.
- Render one fighter on one stage and compare deterministic state traces against the existing TypeScript version where feasible.

**Exit gate:** one local offline round can start and complete with repeatable results.

### Step 4 — Broaden offline parity

Port game modes, complete roster/stages, scenes/menus/HUD, settings/saves, effects, audio/music and optional video handling. Record any unsupported feature explicitly. Measure Linux performance throughout rather than leaving it until the end.

### Step 5 — AppImage CI and later networking

- Build a Linux x86_64 AppImage from the native Bevy app.
- Test the AppImage in CI **without any ROM, disc image, arcade dump, or converted game content**.
- Use software rendering/virtual display only where supported and fail CI on startup/panic/runtime-loading errors; upload logs with the artifact.
- Upload the ROM-free AppImage as a workflow artifact first. Keep release publication in a separate, explicit workflow after build and smoke-test pass.
- The user will test the generated AppImage locally with their own game data; never ask the user to upload their ROM into CI or the repository.
- Implement native online transport after offline behaviour is reliable. The current browser WebRTC facilities are not a drop-in native Bevy capability.

## 6. Legal/data and repository constraints

Follow `AGENTS.md` and the repository's existing policy:

- Never commit or publish game disc images, arcade ROMs/board dumps, executables from the game, extracted/decoded/converted copyrighted game data, screenshots/recordings of game content, or any build artifact containing that content.
- Tests and CI should use synthetic fixtures; tests that genuinely require local user data should skip cleanly when it is absent.
- Keep local generated content in the repository's ignored/local directories as already documented.
- Do not work on bypassing copy protection, region locks or arcade security.
- Keep claims factual; do not state legal conclusions as facts.

## 7. Scope and non-goals at handoff

Completed:
- Reviewed repository tree and major engine-facing modules.
- Added `audit.md` with inventory, migration estimate, risk breakdown, recommended target architecture, milestone plan, and AppImage/no-ROM test expectations.
- Opened PR #1 for documentation review.

Not done:
- No Bevy/Cargo project has been created.
- No TypeScript has been ported to Rust.
- No rendering, audio, input, networking, or gameplay code has changed.
- No Linux workflow or AppImage has been created or tested.
- No ROM or game assets were used for CI testing.

## 8. First prompt to use when resuming

> Resume the DriftEngine-to-Bevy migration in `rakmahefa/driftengine-tekken3`. Read `SESSION_HANDOFF.md` and `audit.md`, verify the current GitHub PR/branch state, and continue from the next unfinished step. Keep Linux first, start with a native Bevy no-ROM proof of concept, and do not bundle or request game data. Do not repeat the initial audit unless the repository changed materially.
