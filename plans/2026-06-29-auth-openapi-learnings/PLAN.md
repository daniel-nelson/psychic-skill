# Process auth + OpenAPI learnings (2026-06-29) — into the skill

Source: `WHAT_I_LEARNED_ABOUT_PSYCHIC_20260629.md` (4 numbered points from a ToS consent-gate build on BearBnB). Fold the verified, missing pieces into the skill; archive the file when done.

## Status

- ✅ 1. Cross-cutting authz gate on the shared auth base; exempt bootstrap endpoints structurally — done (66e2891)
- [ ] 2. `@BeforeAction` `only`/`except` are action-name filters; no per-subclass skip of an inherited hook — pending
- [ ] 3. `forbidden(message)` / `HttpError` JSON-stringifies the body; error responses carry no schema, so a marker is generated-neutral — pending
- [ ] 4. Per-operation error-response set is uniform across auth levels; relocating a controller is generated-neutral — pending
- [ ] 5. Removing default OpenAPI responses via `omitDefaultResponses` (user's steer on #4) — pending
- [ ] 6. Release: VERSION 0.52.0 + CHANGELOG, archive source file — pending

Status legend: ⬜ pending · 🟦 in progress · ✅ done · ⏸ deferred · 🚫 dropped

## Pre-flight findings

- **This repo IS the psychic-skill.** Learnings get folded into the skill markdown directly; we do NOT write a new `WHAT_I_LEARNED_*` file here (this is the consumer side). The source file is archived to `~/Documents/processed-psychic-learnings/` at the end. (CLAUDE.md: never invoke psychic-skill on this repo; memory `feedback_process_learning_files`.)
- **Release process (CLAUDE.md, required every PR):** VERSION bump + matching CHANGELOG entry under a new `## <version> — <date>` heading. Minor bump for new guidance. `pnpm psy` command style in examples. Verify the ecosystem baseline block before finalizing. Don't invent stale-guidance contrast prose.
- **Don't trust the file; verify each point against `~/work/dream_and_psychic`** (psychic source) and translate examples to BearBnB (the file already uses BearBnB: Guest/Host/Visitor, ToS gate).
- **Verification commands:** none automated (skill is markdown + bash, no `package.json`). Correctness is verified per item against psychic source; final check that VERSION/CHANGELOG are consistent and any new internal anchor links resolve.
- **Source pointers already located in pre-flight:**
  - `omitDefaultResponses` (boolean) — `psychic/src/openapi-renderer/endpoint.ts:99,165,844-856,1468-1472`; default set `DEFAULT_OPENAPI_RESPONSES` (400/401/403/404/409/500) in `src/openapi-renderer/defaults.ts:7`; 422 comes from config-level `defaults.responses`. Also a controller-level `openapiConfig.omitDefaultResponses`, and `omitDefaultHeaders`, `defaultResponse`.
  - `@BeforeAction` / `ControllerHook.shouldFireForAction` (name-based `only`/`except`) — verify in `psychic` dist `hooks` + `decorators`.
  - `forbidden(message)` → `HttpStatusForbidden(message)`, `HttpError.data` JSON-stringified body; `Forbidden` response component is description-only — verify in `psychic/src/error/http/*` and the openapi default components.
  - Current skill coverage gaps confirmed: `omitDefaultResponses` absent everywhere; `only`/`except`+no-skip absent; `forbidden(message)` body/marker absent; uniform-response-set/relocation-neutral absent. openapi.md:81 mentions defaults exist but not omission.

## Open questions

- Q-scope: process all four points (items 1–5), or narrow to just the OpenAPI / #4 work? (User led with "/psychic-learning" + a note on #4.)

## Verification

- (not yet run)

## Spinoffs

- (none yet)

---

## Item details

### 1. Cross-cutting authz gate on the shared auth base; exempt bootstrap endpoints structurally

**Status:** ✅ done — commit `66e2891`

**Artifacts:** `controllers.md` — new `### Cross-Cutting Authorization Gates` subsection inserted after the Key Principles block (before `### Verifying the Hierarchy`). 43 insertions.

**Discoveries / notes:**
- Verified against source before writing: `shouldFireForAction` is name-based (`psychic/src/controller/hooks.ts:27-33`); `@BeforeAction` inherits via spread copy of `controllerHooks` and dedups by `methodName` (`decorators.ts:38-42`), so a same-method-name redeclaration in a descendant is a no-op and ancestor hooks come first. Item 1's placement/inheritance claims hold.
- The "why it holds" (no-skip mechanism) is cross-referenced via a forward anchor link `[\`@BeforeAction\` scoping](#beforeaction-scoping)` — that subsection is built by **item 2** in this same PR. Link resolves once item 2 lands; if item 2 titles its heading differently, fix the anchor there.
- Routing reuse: linked the existing `#routing-controllers-when-directory-names-dont-map-to-urls` section rather than duplicating the `controller:` example, per spec.
- Did NOT bump VERSION / CHANGELOG (item 6 handles release).

**Intent.** Document where a cross-cutting authorization precondition lives and how to exempt the endpoints that can't be subject to it, framed as the intentional architecture that controllers.md:5 already promises ("read the base controller's `@BeforeAction`s to know the subtree's rules").

**What to write (controllers.md, attached to the auth-architecture section near line 5):**
- A cross-cutting authz precondition that applies to every authenticated surface (accepted-current-ToS, completed-onboarding, active-subscription, verified-email — keep it general, not ToS-only) goes in ONE `@BeforeAction` on the shared `AuthedController`, declared after `authenticate` so `currentUser` is populated. All authed namespaces inherit it symmetrically.
- Exempt the endpoints that clear the precondition (records consent / completes onboarding) and the not-yet-cleared probe (`/me`) STRUCTURALLY: re-parent to `MaybeAuthedController` and self-guard (`if (!this.currentUser) return this.unauthorized()`). Route to a clean URL via the existing `controller:` pattern — reference controllers.md:88–99, do not duplicate the example.

**Decisions (decide-and-disclose):**
- Framing = intentional guarantee, not limitation-workaround. The no-skip rule (item 2) is *why* the subtree's rules are knowable in one place, with no hidden overrides lower down. (User direction, this turn.)
- Keep item 1 to the architecture; the no-skip mechanism is item 2, the `forbidden(message)` marker is item 3. Cross-reference rather than retell the file's bundled ToS narrative. (Authority: existing skill is organized by reusable fact.)
- Reference the existing clean-URL routing example (controllers.md:88–99) instead of adding a new `/me` routing snippet. (User confirmed it already exists.)

**Acceptance:** controllers.md teaches the gate-placement + structural-exemption rule generally, tied to the existing auth-architecture promise, with no duplicated routing example. Verified: hooks inherit ancestor→descendant in declaration order; `shouldFireForAction` is name-based (`psychic/src/controller/hooks.ts:27`, `decorators.ts:35-40`).

### 2. `@BeforeAction` `only`/`except` are action-name filters; no per-subclass skip

**Status:** pending — discussed

**Intent.** A short, findable "BeforeAction scoping" subsection in controllers.md, cross-linked from item 1.

**What to write:**
- `@BeforeAction({ only, except })` filters by action method NAME (`ControllerHook.shouldFireForAction`), not by controller. `except: ['create']` suppresses the hook for every descendant action named `create`, not for one controller. Small example.
- There is no `skipBeforeAction` and no per-subclass override: descendants inherit the hook, and redeclaring a `@BeforeAction` with the same method name is a no-op (ancestor's hook, including its `only`/`except`, wins). To vary auth for a subtree, re-parent it (visible in the directory tree) — this is what makes the base controller's hooks authoritative for the whole subtree (ties back to item 1).

**Decisions (decide-and-disclose):**
- Separate subsection rather than woven into item 1. `only`/`except` is a general capability an agent searches for on its own. (User, Q2.)

**Acceptance:** controllers.md has a `BeforeAction` scoping subsection covering `only`/`except` (name-based) and the no-skip/no-override guarantee, cross-linked from item 1. Verified: `psychic/src/controller/hooks.ts:27-33` (name-based filter), `decorators.ts:35-40` (inherit via spread, dedup by methodName).

### 3. `forbidden(message)` JSON-stringifies the body; marker, and its typed-enum upgrade

**Status:** pending — discussed

**Intent.** Document, near the response-helpers list (controllers.md:584), two layers for distinguishing same-status errors.

**Layer 1 — the marker pattern (untyped, zero-config).** `this.forbidden(msg)` / `unauthorized(msg)` / etc. throw an `HttpError` whose `data` arg is JSON-stringified as the response body (`forbidden('not_your_place')` → body `"not_your_place"`). The default `Forbidden`/`Unauthorized`/etc. response components are description-only (no `content`/schema), so the marker is invisible to the generated spec and client — a runtime discriminator for two same-status causes with no schema change or regen. The frontend switches on `err.response.data === 'not_your_place'`, an untyped string.

**Layer 2 — the typed-enum upgrade.** The body stays a string; attach an `enum` so the generated client types it as a literal union instead of bare `string`. Declare it as a runtime schema shorthand (NOT a TS type). Caveat to state plainly: declaring the response only changes the spec + generated types. It does not change runtime behavior or enforce the enum. You still implement the `forbidden(...)` throw and keep the thrown value in sync with the declared `enum` by hand; Psychic won't flag drift.

**Scope rule — where the typed response lives depends on where the cause is raised:**
- An *action-specific* cause (e.g. `not_your_place`, only raised inside that one Place action) → per-action `@OpenAPI` `responses` override. Overrides the default 403 `$ref` for that operation (verified: `parseResponses` sets the declared status first; defaults only fill missing keys):
  ```ts
  responses: { 403: { type: 'string', enum: ['not_your_place'], description: '…' } }
  ```
- A *cross-cutting* cause raised on a shared base `@BeforeAction` (e.g. `terms_of_service_required` on `AuthedController`, returnable by every authed endpoint) → document ONCE by redefining the shared response component at conf level (`defaults.components.responses.Forbidden`), NOT per-action. The default 403 `$ref: '#/components/responses/Forbidden'` then resolves to the typed body across the spec. See item 5 / openapi.md for the mechanism. Per-action would falsely imply only that operation returns it and force repetition on every gated endpoint.
- Honesty caveat: conf `defaults` (including a redefined component) apply to the whole spec, not precisely "authed endpoints only." To scope a marker to the authed surface, give those controllers their own named spec via `openapiNames` and redefine `Forbidden` only there; otherwise public endpoints that default a 403 also advertise the marker body.

**Decisions (decide-and-disclose):**
- Fold both layers (user, Q3). Marker first (zero-config escape hatch), then the typed upgrade with the spec-vs-runtime hand-sync caveat.
- Add the scope rule (per-action vs conf-level) from the cross-agent conversation. A shared-base marker is documented at spec level, not per-action. Ties item 3 → item 5 → openapi.md conf-level config.
- Exclude the app-local `pnpm psy sync` 422-drift toolchain issue; it's environmental, not Psychic behavior.

**Acceptance:** controllers.md documents the marker pattern and the typed `enum` upgrade with the spec-only/no-enforcement caveat AND the action-specific-vs-cross-cutting scope rule (cross-ref openapi.md conf-level `defaults.responses`), BearBnB examples. Verified: `controller/index.ts:1238,1269-1270`; `error/http/index.ts:7-12`; `openapi-renderer/defaults.ts:88-110`; `endpoint.ts:1389-1396`; `dream/src/types/openapi.ts:88`; conf-level `defaults.responses` at openapi.md:69.

### 4. Relocating a controller (same route) is generated-neutral

**Status:** pending — discussed

**Intent.** Short standalone note (openapi.md or controllers.md routing). Fact (a) of the agent's #4 moved to item 5; only fact (b) remains here.

**What to write:** the OpenAPI spec is keyed by URL path + HTTP method, with no controller class name and no `operationId` emitted (`endpoint.ts:259-271`). So moving a controller between auth bases (or renaming it) while keeping its route produces zero diff in `openapi.json`, and the downstream-generated SDK function names track the path, not the class. Practical payoff: a controller relocation (e.g. the structural exemption in item 1) is safe to make without regenerating the client.

**Decisions (decide-and-disclose):**
- Keep as a short note; fact (a) merged into item 5 (user, Q4).

**Acceptance:** a brief generated-neutral-relocation note exists, cross-referencing item 1's structural exemption. Verified: no `operationId` in the renderer; path object keyed by `computedPath` + method.

### 5. Default responses are uniform; make them honest via `omitDefaultResponses`

**Status:** pending — discussed

**Intent.** The important one (user's steer). openapi.md, cross-ref `@OpenAPI` options in controllers.md.

**Motivation (fact 4a).** Every operation gets the same default error-response set (`DEFAULT_OPENAPI_RESPONSES` = 400/401/403/404/409/500, plus config-level additions like 422), merged uniformly regardless of the controller's auth base. The auth base never adds or tightens responses. So an operation that genuinely can't return some default (a truly public GET that never 401/403s) still advertises it unless you intervene.

**Capability — precedence and the levers (verified `endpoint.ts:795-874`, `app.ts:85-97`):**

Per-status precedence: per-action `@OpenAPI responses[s]` > conf `defaults.responses[s]` > framework `DEFAULT_OPENAPI_RESPONSES[s]`. Defaults fill a status only `if (!responseData[key])` (line 858); conf merge is a shallow per-status spread `{ ...DEFAULT_OPENAPI_RESPONSES, ...defaults.responses }` (line 852).

- **Override one status, keep the rest** — declare that status in `responses` (per-action) or `defaults.responses` (conf-level, applies spec-wide). Your status wins; every other default remains. No omit needed.
- **Reshape a *shared* response once (the cross-cutting case)** — redefine the component the default `$ref`s point at: conf `defaults.components.responses.Forbidden` (or `.Unauthorized`, etc.) with a body schema/enum. The assembler emits `{ ...DEFAULT_OPENAPI_COMPONENT_RESPONSES, ...defaults.components.responses }` (`app.ts:92-95`, your key replaces the whole component value), so every defaulted 403's `$ref` resolves to the typed body — no per-action repetition, no lost `$ref`. Can also park the schema under `defaults.components.schemas.*` and `$ref` it. Scope to the authed surface with a named spec (`openapiNames`); conf defaults otherwise span the whole spec.
- **Remove one status, keep the rest** — no direct mechanism. Set `omitDefaultResponses: true` (boolean, all-or-nothing) and re-list the keepers.
- **Deep-merge within one status** — NOT supported. Declaring a status (or redefining a component) defines its whole entry; you can't keep the default and append a field. `defaultResponse` (singular) augments only the primary/success response's description.
- **Negative finding (state it):** `omitDefaultResponses` / `omitDefaultHeaders` are NOT conf-level (`psy.set('openapi', ...)`) options. They live only on the per-action `@OpenAPI` decorator and on a controller's static `openapiConfig` getter (`endpoint.ts:161-168`, `controller/index.ts:1599-1602`). The spec-wide way to omit is per-controller `openapiConfig`, not conf.

**Decisions (decide-and-disclose):**
- Lead with the uniformity motivation (fact 4a), then the capability (user, Q4/Q-scope). Keeps "why" before "how".

**Acceptance:** openapi.md explains the uniform default set and how to make per-operation responses honest (`omitDefaultResponses` all-or-nothing + re-add; per-status override via `responses`; controller-level key). BearBnB example (public endpoint dropping 401/403). Verified against `endpoint.ts` and `defaults.ts:7-26`.

### 6. Release: VERSION + CHANGELOG, archive source

**Status:** pending

Bump VERSION to 0.52.0 (minor — new guidance). Add a `## 0.52.0 — 2026-06-29` CHANGELOG entry grouped by Added/Changed covering whatever items 1–5 actually landed. Re-verify the ecosystem baseline block is current. Archive `WHAT_I_LEARNED_ABOUT_PSYCHIC_20260629.md` to `~/Documents/processed-psychic-learnings/`. Open the PR from a new branch off main.
