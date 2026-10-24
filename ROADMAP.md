# Roadmap

Status of the Master V web client and planned work. This file replaces the unchecked list that used to live in the README, and every item was re-checked against the current code (see [Disposition of the previous roadmap](#disposition-of-the-previous-roadmap)).

- **Scope:** this repository is the frontend only. Items that need backend changes are marked **Backend dependency**; the backend is not in this repository, so those are prerequisites outside this codebase.
- **Priorities:** **P0** blocks reliable delivery or safety; **P1** completes or corrects product behavior; **P2** improves maintainability and polish.
- **Evidence:** "Completed" means the implementation exists in the repository. It does not claim test coverage (there are no automated tests).
- Architecture details and weaknesses referenced below: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). Overview: [`README.md`](README.md).

---

## Completed

Evidence is the code location. These form the current product baseline.

| Milestone | User / business value | Evidence |
| --- | --- | --- |
| Authentication: login, 3-step registration, forgot and reset password | Users can create accounts and recover access | `src/Pages/Auth`, `src/Components/Auth`, `src/Redux/Slice/Auth.js` |
| Role-based navigation (`Personal`, `Gold`, `Company`, `SubUser`) | Each account type sees only relevant screens | `src/Hooks/useValidateRole`, `src/App.jsx` |
| Project search with extensive filters (section, city, area, type, bedrooms, finishing, delivery, budget, payment plan, compound, developer, benefits, consultants) | Core value: find matching projects quickly | `src/Components/Data/Sidebar/FilterSideBar`, `src/Utility/BodyHelper.js` |
| Project detail: phases, payment plans, price lists with computed down payment and installment, contacts, files, map | Compare and present deals | `src/Components/Data/_ItemInfo`, `src/Components/Data/View` |
| WhatsApp integration (QR session, send messages/files, contact actions) | Share project information with clients from the app | `src/Components/Settings/QrCode`, `src/Common/WhatsAppSender` |
| Commission tracking for Gold users (request, edit, delete, files, due/collected lists, calculator) | Track earnings per deal | `src/Pages/Sales`, `src/Components/Sales` |
| Sub-user management for Company accounts with city/section scoping | Brokerage owners manage a sales team | `src/Components/Dashboard/Users` |
| Real-time notifications and user/session updates over Socket.IO | Live updates without refresh | `src/Utility/WebSocketController.js`, `src/Components/_WebSocketEvents` |
| English/Arabic localization with RTL; light/dark mode; responsive layouts | Arabic-speaking users and mobile use | `src/Lang`, `src/Layout/DataLayout`, `src/Utility/ChakraTheme` |
| Offline/server-down banner; subscription-expiry banner | Clear status and renewal prompts | `src/Common/OfflineMessage`, `src/Common/ExpiredMessage` |
| Stale-build detection and cache-busting build output | Users pick up new releases | `src/Components/_CheckBuildVersion`, `vite.config.js` |
| Route-level code splitting; SEO metadata, JSON-LD, robots, sitemap; web app manifest; Apache SPA config | Faster first load and basic discoverability/installability | `src/App.jsx`, `index.html`, `public/` |

---

## In Progress

There are no open branches and the history has four commits, so no active work can be verified. The item below is **partially implemented** and is the natural next feature.

### ROI analysis modal — P1

- **Purpose / value:** show investors the return on a unit (ROI, ROE, IRR, NPV, leverage, payback period) so brokers can justify a purchase.
- **Current state:** modal, buttons, labels (`DATA_PAGE.LABELS`) and cards exist; all metric values are rendered as `-` and a previous set of sample numbers is commented out. `Public.ROI` holds defaults for appreciation and discount rate.
- **Technical work:** define the formulas and inputs (price at delivery, appreciation before/after delivery, rent, installment schedule, discount rate); implement them as pure, unit-tested functions; wire the cards (`TopCards`) and table (`ROIContent`); validate inputs.
- **Dependencies:** product definition of formulas and default assumptions; possibly backend-provided defaults (**Backend dependency**, optional).
- **Definition of done:** for a selected unit, all metrics render from computed values; changing inputs updates results; formulas are covered by unit tests with documented examples; no `-` placeholders remain except for genuinely missing data.

---

## Planned

### P0 — Delivery reliability and safety

#### 1. Fix the production build on case-sensitive systems
- **Purpose / value:** the build is the basis for any CI or Linux deployment.
- **Technical work:** `vite.config.js` references `src/lang`, but the folder is `src/Lang`; correct the path (one word), confirm `npm run build` and `npm run preview`.
- **Dependencies:** none.
- **Definition of done:** `npm ci && npm run build` succeeds on Linux; output contains `Lang/ar.js` and `Lang/en.js` under the build-id folder.

#### 2. Continuous integration
- **Purpose / value:** catch regressions before release; showcase engineering discipline.
- **Technical work:** GitHub Actions workflow running `npm ci`, `npm run lint`, `npm run build` (and tests once they exist) on pull requests and `main`.
- **Dependencies:** item 1.
- **Definition of done:** workflow is green on `main`; failing lint errors or build break fail the PR. (Today lint reports 0 errors and 7 warnings.)

#### 3. Automated tests for business logic
- **Purpose / value:** pricing, commission and request-building logic is the highest-risk code and has no tests.
- **Technical work:** add Vitest; extract pure functions where logic is embedded in components (commission calculation, installment/down-payment math, `ExtractBulks`, `HandlePayPlanDiscount`, `BodyHelper.Data`, `Validation.*`); add reducer tests for `Auth`/`Filter`; add a `test` script.
- **Dependencies:** item 1 for CI; sample expected values from the product owner.
- **Definition of done:** `npm test` passes locally and in CI; each listed function has cases for normal, edge (zero, empty, malformed) input.

#### 4. Verify and document server-side authorization
- **Purpose / value:** route guards, role gating and city entitlement filtering run in the browser and can be bypassed; the API must enforce them.
- **Technical work:** review API behavior for `Data`, `SubUser`, `Sales`, `Cdn` and Detail endpoints against each role; document the contract; consider moving entitlement filtering to the server and reevaluating token storage (`localStorage`).
- **Dependencies:** **Backend dependency** (access to API owners/specification).
- **Definition of done:** a written matrix of role vs. endpoint and verified behavior; any client-only enforcement is either moved server-side or explicitly accepted and recorded.

### P1 — Product correctness and completion

#### 5. Resolve the fixed compound filter
- **Purpose / value:** ensure results reflect the user's compound selection.
- **Technical work:** `BodyHelper.Data` sends `DataCompoundId: [1]` while the mapping from `Filter.DataCompoundId` is commented out; confirm intended behavior, then restore or remove the compound filter UI.
- **Dependencies:** product/backend confirmation.
- **Definition of done:** selecting compounds changes the request body and the results; covered by a `BodyHelper` unit test.

#### 6. Remove `eval` and hard-coded commission/tax constants
- **Purpose / value:** safer and auditable money calculations.
- **Technical work:** replace `eval` in `src/Pages/Sales/index.jsx` and `PriceList/index.jsx` with direct arithmetic; move rates (14%, 5%, 22%, 85%, 1.14 divisor) to named, documented constants or server-provided settings.
- **Dependencies:** item 3 (tests lock behavior first); product confirmation of the formulas.
- **Definition of done:** no `eval` in `src`; calculator output identical to previous results for a reference set of inputs.

#### 7. Project comparison
- **Purpose / value:** side-by-side evaluation of projects/payment plans (a nav icon and `Comparison` label already exist; the route is commented out in `RoutingManager.js`).
- **Technical work:** route and page, selection mechanism from the project list, comparison table for price, delivery, payment plan, finishing.
- **Dependencies:** item 5; may need an API endpoint for multi-project fetch (**Backend dependency**, optional).
- **Definition of done:** user selects 2–4 projects and sees a comparison; route is role-aware and appears in navigation and the manifest/sitemap only if public-safe.

#### 8. Granular roles and permissions
- **Purpose / value:** the current model has four role keys plus city/section scoping; larger brokerages need finer control.
- **Technical work:** replace role-key switches with permission checks; UI for assigning permissions.
- **Dependencies:** item 4; **Backend dependency**.
- **Definition of done:** at least one new permission-controlled capability works end to end and is enforced by the API.

### P2 — Maintainability, quality and polish

#### 9. Consolidate the two Data layouts and remove legacy code
- **Value:** lower maintenance cost. **Work:** measure usage of "Last Version" vs "New Version", migrate remaining users, delete `Pages/Data/1`, `_OldVersion`, `Sidebar/_V1`, `Navbar/OldVersion`, "Companies V1" if obsolete. **Dependencies:** usage data or product decision. **Done when:** one layout remains, no unreferenced legacy folders, lint clean.

#### 10. Refactor orchestrator components and Redux leftovers
- **Value:** readability and testability. **Work:** extract hooks from `Pages/Data/index.jsx` and `Pages/Sales/index.jsx`; remove unused `Filter` reducer actions (`SetTableSort`, `NavigtePage`, `ChangeFilterLimit`, `ChangeFilterType`); fix socket listener cleanup in `WebSocketEvents` (named handler references). **Dependencies:** item 3. **Done when:** pages are smaller, behavior unchanged under tests, `Socket.Off` removes the registered handlers.

#### 11. Type safety for API contracts
- **Value:** catch payload mismatches early. **Work:** adopt TypeScript incrementally (or JSDoc types) starting with `Redux/Slice` and `BodyHelper`. **Dependencies:** item 3. **Done when:** API request/response shapes for Data, Auth and Sales are typed and checked in CI.

#### 12. Accessibility and SEO audit
- **Value:** usability and discoverability. **Work:** keyboard/focus/ARIA review of filters and modals, contrast in both color modes, `lang`/`dir` handling; regenerate `sitemap.xml` with public URLs only (remove `/Comparison` and private routes), set page-level titles/descriptions. **Dependencies:** none. **Done when:** an audit report exists with fixes applied to critical issues; sitemap lists only existing public URLs.

#### 13. Observability and performance baseline
- **Value:** know when production breaks and how fast it is. **Work:** add error reporting for the error boundary and failed requests; record bundle size and Core Web Vitals baselines; revisit `chunkSizeWarningLimit: 5120` (set to 5 MB in `vite.config.js`). **Dependencies:** choice of tooling. **Done when:** errors reach a monitoring tool and a documented performance baseline exists.

#### 14. Privacy and localization clean-up
- **Value:** consistency and privacy. **Work:** stop calling `api.ipify.org` from the browser (let the API derive the IP); move hard-coded English strings (for example in `NetworkErrorBoundary`, `CheckBuildVersion`) into `src/Lang`; set a descriptive `package.json` name. **Dependencies:** **Backend dependency** for the IP change. **Done when:** no third-party IP lookup in the client; all user-visible strings are translated.

---

## Future Considerations

Ideas that have no code foundation yet. They stay unprioritized until product needs are confirmed.

| Idea | Value | Notes / prerequisites |
| --- | --- | --- |
| Analytics and reporting dashboards | Insight for company owners (team activity, pipeline) | The current `/Dashboard` manages users, not analytics; needs backend metrics |
| Saved searches and alerts | Re-engage users when matching projects change | Extends the existing filter model and notification channel |
| Service worker and offline caching | True PWA behavior (the manifest exists, no service worker) | Careful with the project's no-cache hosting policy and build-id strategy |
| Native mobile app | Reach mobile-first users | Evaluate the responsive web/PWA first |
| Integrations with other real estate platforms | Broader data sources | Needs partner APIs and a backend |
| Recommendation engine (including AI-assisted) | Suggest suitable projects | Needs data, privacy review and a backend |
| Full multi-tenant model | Isolated company workspaces | A Company/SubUser hierarchy exists; true tenancy is a backend design |
| Document management | Organized contracts and price lists | File upload/viewing exists for contracts and price lists; no document model |

---

## Disposition of the previous roadmap

| Previous item | Current state in code | Where it went |
| --- | --- | --- |
| Enhanced analytics and reporting features | Not present; dashboard is user management | Future Considerations |
| Mobile application development | Not present; responsive web and manifest exist | Future Considerations |
| Advanced search and filtering capabilities | Largely delivered (more than 20 filter fields) | Completed; saved searches in Future |
| Integration with third-party real estate platforms | Not present (WhatsApp messaging integration exists) | Future Considerations |
| AI-powered property recommendations | Not present | Future Considerations |
| Multi-tenant support | Partial concept (Company/SubUser scoping) | Planned #8 and Future |
| Enhanced notification system | Basic system delivered | Completed; richer alerts in Future (saved searches) |
| Document management system | File viewing/upload exists, no document model | Future Considerations |
| Advanced user roles and permissions | Four roles plus city scoping | Planned #8 |
| Performance optimizations | Code splitting and hashed output exist; no measurements | Planned #13 |
