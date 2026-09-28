# ADR-0001: Implementation language, toolchain, test runner, CI

**Status:** proposed
**Issue:** #2
**Date:** 2026-09-28

## Context

The [PRD](../PRD.md) does not say what Telivi is written in, and nothing in
[Phase 1](../ROADMAP.md#phase-1--thin-slice-with-stand-ins) can start until
it does. The constraints that matter:

- The contributor runtime runs in a browser tab and on a phone, and "the phone
  and the browser are the same kind of contributor"
  ([PRD § Implementation Decisions](../PRD.md#implementation-decisions)).
  User story 2 adds: from a tab, without installing an app.
- Coordinator, weight ledger, and stake book run wherever the project runs
  them. They never need to be on a phone.
- Phase 1 is stand-ins only: no real training, no real device. The choice must
  not make Phase 2 (real browser compute, then a real phone) a rewrite.
- Phase 2 real compute in a browser means WebGPU, which is now enabled by
  default in Chrome 113+, Chrome for Android 121+, Firefox 141+ (Windows) and
  Safari/iOS 26+ ([web.dev](https://web.dev/blog/webgpu-supported-major-browsers),
  [implementation status](https://github.com/gpuweb/gpuweb/wiki/Implementation-Status)).
  WebGPU is a JavaScript API; any language reaches it through JS glue.
- The PRD wants anyone to join and inspect. Contributor onboarding and a small,
  single toolchain matter more than peak performance in Phase 1.
- [AGENTS.md](../../AGENTS.md#working-an-issue) step 6 needs one named test
  command, and commit scopes are the four modules `runtime`, `coordinator`,
  `ledger`, `stake`.

Not decided here: identity (ADR-0002), proof of work (ADR-0003), model
architecture, how the coordinator reaches a device, stake transfer.

## Options

### A. One language for everything: TypeScript
Browser and phone run the same TypeScript runtime, as a page or an installable
web app in the phone's browser; the server side runs on Node. One compiler,
one test runner, one CI job. Real compute in Phase 2 is reached directly via
WebGPU (or WebGL/CPU fallback) from the same code. Costs: no native-speed CPU
kernels without adding WASM later; a native phone shell (if a web app is ever
not enough) is a Phase 3 question. Rules out nothing: a WASM kernel can be
added behind the runtime interface as a build artifact, not a rewrite.

### B. Compiled core (Rust → WASM) with a thin shell per platform
Ledger, stake, coordinator and the training kernel are one Rust crate compiled
to WASM ([wasm-bindgen](https://rustwasm.github.io/wasm-bindgen/)); a small JS
shell per platform loads it. Wins: memory-safe, fast CPU kernels, one core.
Costs: two toolchains (cargo + a JS bundler) and two test runners from the
first issue; WebGPU still needs the JS layer or [wgpu](https://wgpu.rs/)'s web
backend, so the "thin" shell is not thin; Phase 1 has nothing to compute, so
the cost lands before any benefit; contributor onboarding needs Rust and Node.

### C. Split: server in one language, runtime in another
For example Go or Python for coordinator/ledger/stake, TypeScript for the
runtime. Wins: each side uses its strongest ecosystem. Costs: the interface
between coordinator and runtime (work unit, result, verification payload) is
defined twice and drifts; two CIs, two test runners, two onboarding paths;
Phase 1's stand-in device, which is the one place these meet, becomes the
hardest thing to test; nothing in the PRD needs a server-side language other
than the one the runtime already forces.

## Recommendation

**Option A. Telivi is written in TypeScript, end to end.** The runtime has to
be browser code, and the PRD says the phone is the same kind of contributor,
so the language is effectively chosen on that side; A is the only option that
does not pay for a second toolchain before Phase 2 needs one. B's advantage
(fast kernels) can be added later behind the runtime interface without moving
the ledger, stake book or coordinator; C's advantage does not exist for Telivi.

Concretely, for the first Phase 1 issue to add (nothing is added in this PR):

- **Language and runtime:** TypeScript in `strict` mode, ESM only. Node on the
  current **Active LTS** line (24.x today; 26.x from 2026-10-28), pinned in
  `.nvmrc` and `engines`; bump within one issue when the LTS line changes
  ([Node release schedule](https://github.com/nodejs/release#release-schedule)).
- **Package manager:** npm with
  [workspaces](https://docs.npmjs.com/cli/v11/using-npm/workspaces). It ships
  with Node, so a contributor installs nothing extra. `package-lock.json` is
  committed; CI uses `npm ci`.
- **Test runner:** [Vitest](https://vitest.dev/guide/). TypeScript with no
  build step, workspace-aware, and its stable
  [browser mode](https://vitest.dev/guide/browser/) runs the same tests in a
  real browser via Playwright when Phase 2 replaces the stand-in runtime, so
  the runner does not change between phases. Node's built-in `node:test` was
  considered and rejected only for that reason.
- **Formatter and linter:** [Biome](https://biomejs.dev/) (`biome ci .`), one
  dev dependency that does both, with the recommended rule set.
- **Typecheck:** `tsc -b` over per-package `tsconfig.json` files linked by
  [project references](https://www.typescriptlang.org/docs/handbook/project-references.html).
- **Dev dependencies, and only these:** `typescript`, `vitest`, `@biomejs/biome`,
  `@types/node`. No runtime dependencies in Phase 1.
- **Repo layout:** `packages/<module>/` for `runtime`, `coordinator`, `ledger`,
  `stake`, each with `src/index.ts` as its only public entry and tests in
  `packages/<module>/test/*.test.ts`. Separate packages rather than `src/<module>/`
  because the PRD says each module "hides a hard problem behind a small
  interface": the package boundary is that interface, cross-module imports go
  through `index.ts` only, and Phase 2 bundles `runtime` for a browser without
  dragging the coordinator along. Tests live in `test/` and import the public
  entry, which is how the PRD's
  [Testing Decisions](../PRD.md#testing-decisions) rule (observable behaviour,
  not internals) gets enforced by construction.
- **CI:** `.github/workflows/ci.yml`, one job on `ubuntu-latest`, triggered on
  `pull_request` and `push` to `main`, with `concurrency` cancelling stale runs:
  `actions/checkout` → `actions/setup-node` (`node-version-file: .nvmrc`,
  `cache: npm`) → `npm ci` → `npm run lint` → `npm run typecheck` → `npm test`
  ([workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions),
  [setup-node](https://github.com/actions/setup-node)).
- **Test command:** `npm test` (runs `vitest run` across all workspaces; this
  is the command AGENTS.md § Working an issue step 6 refers to).

## Consequences

- Possible now: #5 (weight ledger), #6 (stake book) and #7 (coordinator +
  stand-in runtime) can start. The first of them to merge adds the root
  `package.json`, workspaces, `.nvmrc`, `tsconfig`, Biome config, the CI
  workflow, and its own `packages/<module>/`; the others add only their
  package. The Phase 1 exit command is a root `npm run demo` script added by
  the coordinator issue.
- Harder: CPU-heavy kernels in Phase 2 will be slower than native until a WASM
  kernel is added; that addition is a build artifact inside `packages/runtime`
  and a new `decision` issue if it needs a second toolchain in CI.
- A native phone shell is not decided. Phase 2 targets the phone's browser
  (WebGPU on Android 12+/Chrome and iOS 26+/Safari); if that proves
  insufficient, wrapping the same runtime is a new ADR, not a rewrite.
- This ADR fixes no wire protocol, storage engine, or identity scheme. Those
  stay under "Decisions not made yet" or in ADR-0002/0003.
