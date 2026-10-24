# Architecture and project structure

This document expands the summary in the [README](../README.md). Everything here was derived from the source code in this repository. The backend (REST API, Socket.IO server, CDN) is **not** part of this repository, so its behavior is described only as far as the client depends on it.

## 1. Runtime composition

```
index.html ──loads──▶ /Config.js (window.Config, per environment, not committed)
     │
     └──▶ src/main.jsx
            NetworkErrorBoundary
              └ BrowserRouter
                  └ Redux Provider (Store.Configration)
                      └ ChakraProvider (Theme.Init)
                          └ App.jsx
                              ├ CheckBuildVersion      (production only)
                              ├ WebSocketEvents        (socket lifecycle, auth/route guard, notifications)
                              ├ OfflineMessage         (when API socket is unreachable or browser is offline)
                              └ <Routes> (lazy pages inside Suspense)
```

### Routes

Defined in `src/App.jsx`; paths and metadata are centralized in `src/Utility/RoutingManager.js` (`RoutingManager.Client`).

| Path | Page | Registered for | Marked `Protected` |
| --- | --- | --- | --- |
| `/` | `Pages/Auth` (login / register / forgot) | everyone | no |
| `/Auth/Reset` | `Pages/Auth/Reset` | everyone | no |
| `/Data` | `Pages/Data` (renders `1/` or `2/` by layout version) | everyone | yes |
| `/Settings` | `Pages/Settings` (Account, WhatsApp connection tabs) | everyone | yes |
| `/Location.Project` | `Pages/Detail` (project by `?P=` and `?Id=`) | everyone | no |
| `/Dashboard` | `Pages/Dashboard` (Users, Companies V1 tabs) | `Company` role only | yes |
| `/Sales` | `Pages/Sales` (commissions) | `Gold` role only | yes |
| `*` | `Pages/Error404` | everyone | n/a |

"Protected" paths are enforced in `WebSocketEvents` by redirecting to `/` when no token is present in storage. Role-gated paths are simply not registered for other roles (they fall through to the 404 route). Both are client-side only.

## 2. Responsibilities by directory

| Directory | Responsibility | Notes |
| --- | --- | --- |
| `src/Pages` | Route-level screens. They own data fetching (RTK Query hooks), orchestration state and error handling (401/403 → logout), and pass props down. | `Pages/Data/index.jsx` is the busiest orchestrator: section/city restoration, request building, entitlement filtering, modal state, CDN loading. |
| `src/Components` | Feature UI grouped by domain: `Auth`, `Data`, `Dashboard`, `Sales`, `Settings`, `Notifications`. Also two always-mounted infrastructure components (`_WebSocketEvents`, `_CheckBuildVersion`). | `Components/index.jsx` is a barrel file that re-exports components. |
| `src/Components/Data` | The core product UI. `Sidebar/FilterSideBar` (one folder per filter), `_DataContent` (result list), `_ItemInfo` (project detail: phases, payment plans, price list, ROI/Pay modals), `View` (layouts, materials, price lists), `Modals`, `ContactsTable`, `_OldVersion` (legacy list). | Folders prefixed `_` are internal building blocks or legacy variants. |
| `src/Common` | Reusable presentational components (inputs, selects, tables, navbar variants, loaders, file viewers, switches, `WhatsAppSender`). | Barrel export in `Common/index.jsx`. |
| `src/Layout` | `DataLayout` (responsive two-column layout that handles RTL and sidebar toggling) and `NetworkErrorBoundary`. | |
| `src/Hooks` | `useValidateRole` (role flags), `useValidateSection` (selected section/city), `useLang` (dynamic import of `Lang/<code>.js`), `useNotify` (Chakra toast wrapper), `useOnline`, `useWindowSize`. | |
| `src/Redux` | `Store.js` (store + `ApiSlice` + API path constants), `Reducers/` (client state), `Slice/` (injected RTK Query endpoints per resource). | `Redux/index.js` re-exports all reducers and endpoint hooks. |
| `src/Utility` | `RoutingManager` (paths, socket events, commands, base URLs), `Helper` (storage, formatting, pricing helpers, constants), `BodyHelper` (builds API request bodies from state), `Validtion` (form validation rules), `StateHelper` (initial form states + generic reducer), `WebSocket` / `WebSocketController`, `ChakraTheme/`. | The filename `Validtion.js` is spelled this way in the code. |
| `src/Lang` | `en.js` and `ar.js` translation dictionaries (about 600 lines each). | Loaded on demand by `useLang`. |
| `public` | Static assets, `manifest.json`, `robots.txt`, `sitemap.xml`, `.htaccess`, images including developer logos named `<DeveloperId>.webp`. | `Config.js` and `buildId.txt` are generated/supplied per environment and gitignored. |
| Repo root | `vite.config.js`, `eslint.config.js`, `.prettierrc.json`, `jsconfig.json`, `package.json`. | There are no test, CI or deployment-pipeline files. |

## 3. How the main modules communicate

### 3.1 HTTP (RTK Query)

```
Component / Page
   └ useXxxQuery / useXxxMutation  (src/Redux/Slice/*.js)
        └ Store.ApiSlice (createApi, reducerPath "MasterConfigration")
             └ fetchBaseQuery  baseUrl = RoutingManager.ApiUrl  (DOMAIN_URL + API_URL)
                  prepareHeaders: Authorization: Bearer <token>, IpAdress, Local (language), Device
```

API paths are constants on `Store.Api`:

| Constant | Path | Used by |
| --- | --- | --- |
| `Auth` | `/public/auth` (`/login`, `/register`, `/pass/forget`, `/pass/check`, `/pass/reset`, `/update`; all `PATCH`) | `Slice/Auth.js` |
| `Data` | `/public/data` (search `PATCH`, info `GET`, `/show` `GET`) | `Slice/Data.js` |
| `Contact` | `/public/contact` | `Slice/Data.js` (`GetContacts`) |
| `Cdn` | `/public/cdn` | `Slice/Cdn.js` |
| `Session`, `Message` | `/public/session`, `/public/send` (WhatsApp session and sending) | `Slice/Session.js` |
| `SubUser` | `/user/sub` | `Slice/SubUser.js` |
| `ProjectSales` | `/public/sales` | `Slice/ProjectSales.js` |

### 3.2 Sockets

```
WebSocketEvents (component)
  ├ RoutingManager.SocketConnection(lang, token) → socket.io client (autoConnect: false)
  ├ wraps it in WebSocketInstance (thin wrapper: On/Off/Emit/Connect/Close)
  ├ emits  Root:Index  → server returns device, user and public reference data (sections, cities, ...)
  └ for each event in RoutingManager.SocketEvents:
        Socket.On(event, data => Controller["<Event><Command>"](data))
        e.g. "Notification" + "Update" → WebSocketController.NotificationUpdate
```

`WebSocketController` is the only place that translates server pushes into Redux actions (`LoginR`, `UpdateUserData`, `SetPublicData`, `SetNotifications`, `SetWpData`, ...) and user-visible effects (toasts, browser notifications). Event names: `Error`, `Expired`, `SubUser`, `Auth`, `User`, `Root`, `Cdn`, `Integration`, `Notification`.

### 3.3 State

| Redux key | Source file | Holds | Persisted to `localStorage` |
| --- | --- | --- | --- |
| `Auth` | `Reducers/Auth.js` | user id, name, role, company, selected cities, access token, expiry | yes (`User`) |
| `Helper` | `Reducers/Helper.js` | language, RTL, layout version, sidebar toggle, loader, server status, IP, expiry-message flag | language and version |
| `Filter` | `Reducers/Filter.js` | search filters (section, cities, ranges, ids) | section and city selections |
| `SubFilter` | `Reducers/SubFilter.js` | filter/sort state for sub-user lists | no |
| `AppData` | `Reducers/AppData.js` | selected project | no |
| `Public` | `Reducers/Public.js` | sections, cities, ROI defaults, notifications | no |
| `Wp` | `Reducers/WhatsApp.js` | WhatsApp connection state | no |
| `MasterConfigration` | `Store.js` | RTK Query cache | no |

`Helper.GetStorage/SetStorage` read and write one JSON blob under the `browserData` key. Keys used in `src`: `User`, `Lang`, `Version`, `Section`, `City`, `Device`, `BuildId`, `IpAdress`, `AllowedNotification`; `WebSocketEvents` also reads a `Notifications` entry (read-notification ids by day).

### 3.4 Data-page pipeline (the core flow)

```
Sidebar filters ──dispatch SetFilter──▶ Redux Filter
                                           │
Pages/Data: BodyHelper.Data(Filter) ───────┘  (maps UI selections to API body)
      │
      ▼
useGetDataQuery (PATCH /public/data)
      │
      ▼
ProcessData (client-side entitlement filter for non-Personal roles)
      │
      ├▶ AppDataContent (project list)  ──select──▶ Redux AppData.SelectedItem
      └▶ ItemInfo (useGetInfoQuery by compound + section)
              ├ PhasesButtons / PhasInfo
              ├ PaymentPlan → PriceList (price, down payment, monthly installment)
              └ Modals: Pay, ROI (partial), Map, Actions (share, files)
```

## 4. Where specific things live

| Concern | Location |
| --- | --- |
| Pricing and payment-plan math | `src/Utility/Helper.js` (`HandlePayPlanDiscount`, `ExtractBulks`, `NumberWithCommas`) and `src/Components/Data/_ItemInfo/PriceList/index.jsx` |
| Commission calculation | `src/Pages/Sales/index.jsx` (`CalculateCommisionRate`) |
| Form validation | `src/Utility/Validtion.js` |
| API request body shaping | `src/Utility/BodyHelper.js` |
| Role flags | `src/Hooks/useValidateRole` |
| Translations | `src/Lang/en.js`, `src/Lang/ar.js` |
| Theme and colors | `src/Utility/ChakraTheme`, `src/index.css` |
| Build and chunk naming | `vite.config.js` |
| Server rewrite, redirect, cache rules | `public/.htaccess` |
| Linting and formatting rules | `eslint.config.js`, `.prettierrc.json` |
| Tests | none |
| CI/CD | none |

## 5. Architectural weaknesses and improvement opportunities

These are documented, not changed, as requested. Priorities refer to [`ROADMAP.md`](../ROADMAP.md).

| # | Observation | Evidence | Suggested direction | Priority |
| --- | --- | --- | --- | --- |
| 1 | The production build is not portable to case-sensitive file systems. | `vite.config.js` uses `src/lang`; directory is `src/Lang`; `vite build` fails with `ENOENT` on Linux. | Correct the path; add CI that builds on Linux. | P0 |
| 2 | Authorization boundaries are in the client. | Role-gated routes in `App.jsx`; entitlement filtering in `Pages/Data/index.jsx`; token in `localStorage`. | Confirm and document that the API enforces roles and city entitlements; consider memory/HttpOnly-cookie token storage if the backend supports it. | P0 |
| 3 | No automated tests around pure business logic. | No test runner in `package.json`. | Extract pricing, commission and request-body logic into pure functions and cover them (Vitest). | P0 |
| 4 | Business constants and `eval`. | `Pages/Sales/index.jsx` and `PriceList/index.jsx`. | Replace `eval` with plain arithmetic; move rates to named constants or server-provided config. | P1 |
| 5 | Request body contains a fixed compound filter. | `DataCompoundId: [1]` in `BodyHelper.Data`, mapping commented out. | Confirm the intended behavior and restore or remove. | P1 |
| 6 | Dead or inconsistent Redux code. | `Filter.js` initial state uses `Limit`, `Page`, `Sort`, but `SetTableSort`, `NavigtePage`, `ChangeFilterLimit`, `ChangeFilterType` write lowercase keys that `BodyHelper` does not read, and they are not used by any component. | Remove or align the unused actions. | P2 |
| 7 | Two parallel Data UIs and legacy folders. | `Pages/Data/1` and `2`, `Components/Data/_OldVersion`, `Sidebar/_V1` and `_V2`, `Navbar/OldVersion` and `NewVersion`. | Retire the old layout once usage data allows, then delete legacy components. | P2 |
| 8 | Large orchestrator components. | `Pages/Data/index.jsx` and `Pages/Sales/index.jsx` mix fetching, derived state, side effects and modal state. | Extract custom hooks (for example `useDataFilters`, `useSalesForm`). | P2 |
| 9 | Socket listener cleanup is ineffective. | In `WebSocketEvents`, `Socket.Off` is called with new inline arrow functions, which do not match the registered handlers. | Keep named handler references and remove them in cleanup. | P2 |
| 10 | Inconsistent conventions. | PascalCase variables and file names; typos in identifiers (`Validtion`, `Configration`, `IpAdress`, `NavigtePage`); some hard-coded English strings. | Adopt a naming convention going forward; rename incrementally with tests. | P2 |
| 11 | Static assets documentation drift. | `sitemap.xml` includes `/Comparison` and private routes; `package.json` name is `app`. | Regenerate the sitemap with public URLs only; set a descriptive package name when branding is final. | P2 |
| 12 | Third-party IP lookup. | `api.ipify.org` call in `WebSocketEvents`. | Let the backend derive the client IP. | P2 |
