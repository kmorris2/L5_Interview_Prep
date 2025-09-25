# QR App Generator Tech interview 


## QR App Generator

- Had to upgrade MUI from v4 to v11 so then I could migrate to Canvas v7
- Added with the team’s consent emotion React (what is emotion react?)
- They were behind on Praxis versioning so updated that as well from v20 to v22

5-minute opening narrative (use this to set context)
* Problem & constraints: “I owned the migration of a React app from MUI v4→v5, then to a new component library. Goals: remove JSS, standardize theming, unblock React 18, keep visual parity, and stabilize CI.”
* Key choices:
    * Picked ThemeProvider at root, replaced v4 imports, removed adaptV4Theme (deprecated), moved to Emotion (then to new lib).
    * Replaced Enzyme with React Testing Library for user-centric tests.
    * Created data-testid where needed and codemods to bulk-update variants.
* Risks & trade-offs:
    * Peer-dep conflicts (React 17/18, @mui/styles), solved via package graph cleanup (uninstall v4, lock React 18, install Emotion) and avoiding --legacy-peer-deps in CI.
    * Temporary UI drift during migration; mitigated with visual checks and story sandbox.
* Results: zero prod downtime, green tests, faster local build, simpler styling surface area, documented runbook.
* Lessons: lockstep upgrades (React + UI), treat tests as contracts, and keep migrations reversible with small PRs.

Likely question areas & crisp answers
1) “Why this framework/library?” (and alternatives)
* Why new UI library: design system alignment, long-term maintenance, theming tokens, performance (lighter runtime CSS), first-class React 18 support.
* Alternatives considered: Staying with MUI (pros: mature ecosystem; cons: peer-dep friction & theming differences with your org’s DS), Tailwind + headless UI (pros: control & perf; cons: more primitives to wire).
* Decision criteria: team skills, accessibility guarantees, SSR behavior, bundle size, theming API, component coverage.

2) Migration strategy & risk control
* Incremental: codemods → imports → theme → components → tests.
* Compat shims: data-testid, minimal wrapper components to reduce churn.
* Rollout: feature flags by route/page; visual regression spot checks; CI gate on unit/E2E.

3) Dependency conflicts (ERESOLVE)
* Method: map peer deps (npm ls react), remove stale v4 packages, pin React/ReactDOM together, add required peers (Emotion), rerun clean install. Avoid --legacy-peer-deps except locally; never in CI.
* Codebase hotspots: package.json, setupTests.js, legacy imports, theming root.

4) Testing approach (why RTL over Enzyme)
* Reason: RTL tests behavior (DOM/user POV), resilient to refactors; Enzyme couples to internals & lacks React 18 support.
* Coverage: happy path (valid URL → QR), error path (invalid URL → Alert), reset path (Retry), accessibility roles/labels.

5) Scale & performance
* Rendering: memoize heavy components (QR generation), debounce input, lazy-load non-critical chunks, prefer CSS vars over JS re-calc for theming.
* Bundles: tree-shake, analyze with source-map-explorer, code-split by route.
* Runtime: avoid prop drilling (context or light state lib); limit re-renders with React.memo, useCallback.

6) Accessibility (you’ll win points here)
* Inputs have labels, alerts use role="alert" (or component ensures it), buttons are focusable when actionable. After submit (valid or invalid), move focus appropriately (e.g., to Retry or to alert).
* Keyboard paths tested in Cypress: Tab → Enter submits; Esc or Retry resets.

7) Security & robustness
* Input validation: stricter URL check (use URL constructor; whitelist schemes).
* Sanitization if rendering user content in future.
* Error boundaries for unexpected render failures.

8) Extensibility prompts you might get
* Multiple QR types (Wi-Fi, vCard): abstract encoder, pass schema → formatter.
* Theming for brands: design tokens + theme objects; no component-level hardcoded colors.
* i18n: externalize strings; RTL layout checks.
* Offline mode: service worker to cache generator.

Concrete artifacts to bring (or describe)
* Before/after dependency graph (React/MUI/emotion/new lib).
* Theme diagram: where ThemeProvider lives; token mapping.
* Test matrix: unit (RTL), E2E (Cypress), lint/TS, bundle check.
* Rollback plan: revert to previous minor; feature flag toggles.

Snappy Q&A you can reuse
Q: How did you ensure Retry is focusable when showing the Alert? A: I set retryDisabled:false in the invalid path and optionally shift focus:

this.setState({ showError: true, retryDisabled: false }, () => {
  this.retryBtnRef?.focus();
});

(Keep a ref on the Retry button; improves keyboard UX.)

Q: How would you validate URLs robustly? A:

const isValidUrl = (v) => {
  try { const u = new URL(v); return ['http:', 'https:', 'otpauth:', 'com.rsa.securid:'].includes(u.protocol); }
  catch { return false; }
}
Then gate state changes on isValidUrl(qRValue).

Q: How did you keep tests stable through the migration? A: Test by role/label/testid, avoid implementation details, and pin regressions with Cypress E2E for critical flows.


React Testing Library vs Enzyme (benefits to emphasize)
* Tests behavior, not internals: RTL queries the DOM like a user (getByRole, getByLabelText) vs Enzyme poking at component instance/state. This reduces brittle tests and aligns with real UX.
* Future-proof with modern React: RTL works with hooks, Suspense, concurrent features; Enzyme has lagged behind React’s faster release cadence and lacks full support for newer APIs.
* Encourages accessible markup: Because you query by role/name, RTL nudges you toward proper labels and ARIA—improving a11y and test stability.
* Less coupled to implementation details: Refactors (split components, change hooks) won’t break RTL tests if behavior stays the same; Enzyme shallow renders often shatter on refactors.
* Community & maintenance: RTL is actively maintained and recommended by the React community; Enzyme’s ecosystem has slowed, adapters can be stale.
* Realistic rendering model: RTL renders to a real DOM (via JSDOM) and discourages shallow rendering, catching integration bugs Enzyme may miss.
* Cleaner async patterns: RTL’s findBy*/waitFor mirrors user-visible async, while Enzyme often relies on manual act gymnastics.


Emotion react library was installed to upgrade from MUI v4 to MUI v5. If it is still in the code, that is because the team still wanted it there. I advised them that it is no longer needed and serves them no use but they kept it. I advised them to remove it in their future work


