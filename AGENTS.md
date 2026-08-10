# PeakURL repository context

This file is the working map for agents and contributors. It records the
architecture and conventions found in the source tree; update it when a
structural or operational behavior changes.

## Product and stack

PeakURL is a self-hosted branded short-link and analytics application. It is a
single repository containing:

- A PHP 8 application runtime and JSON API backed by MySQL/MariaDB.
- A React 19 + TypeScript dashboard built with Vite 8.
- A browser installer and packaged-release front controller.
- Docker Compose development services for the UI, PHP app, MySQL, phpMyAdmin,
  local HTTPS proxy, and packaged-release test site.
- Gettext source/catalog files for dashboard and installer translations.

There are no repository tests or test runner configured. Verification is done
with TypeScript/Vite build, ESLint, PHP_CodeSniffer, PHP syntax checks, Docker
smoke checks, and manual packaged-install testing.

The current source version is read from `.version`. Do not assume the
`package.json` version is authoritative for releases; `scripts/build-release.sh`
uses `.version` first.

## Repository map

### Root and operations

- `README.md`: product overview and user-facing installation overview.
- `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `SUPPORT.md`: project
  policies. Security reports go to `security@peakurl.org`; do not turn a
  security report into a public issue.
- `package.json`: UI/build/lint/release command entry points.
- `vite.config.ts`: Vite aliases, relative build base, dev proxy, and output
  naming.
- `tsconfig*.json`: strict TypeScript project configuration.
- `eslint.config.js`, `.prettierrc.json`, `phpcs.xml`: formatting and lint rules.
- `compose.yaml`: local Docker topology and development defaults.
- `scripts/setup-local.sh`: installs mkcert CA/certs and edits `/etc/hosts`.
- `scripts/build-release.sh`: builds the UI, installs production Composer
  dependencies, assembles `release/peakurl`, zips it, and writes release
  metadata/checksum.
- `scripts/i18n/*.php`: extract POT, update PO files, and compile catalogs.
- `.github/workflows/`: release automation. A release tag must match `.version`.

### PHP application

- `app/public/index.php`: API front controller. Loads Composer, runtime config,
  i18n, CORS/security headers, and `Includes\Application`.
- `app/includes/application.php`: application kernel. Creates `Store`, registers
  every API route, dispatches requests, converts exceptions to JSON, and sends
  headers/cookies/body.
- `app/http/`: request parsing, route matching, JSON envelope, and API errors.
- `app/controllers/`: thin route handlers. Controllers validate/unwrap request
  input only enough to delegate domain work to `Store`.
- `app/store.php`: central persistence/domain facade and dependency container.
  Its implementation is split across traits in `app/traits/`.
- `app/api/`: small reusable raw-query helpers for links, users, and settings.
- `app/includes/peakurl-db.php`: PDO/database abstraction with table-prefix,
  identifier, prepared-query, CRUD, transaction, and schema helpers.
- `app/includes/connection.php`: runtime PDO connection and simple database
  metadata operations.
- `app/includes/runtime-config.php`: merges config sources and normalizes paths,
  database values, cookies, environment, and debug behavior.
- `app/includes/constants.php`: canonical config keys, API base path, defaults,
  schema version, settings keys, and install-only keys.
- `app/includes/functions.php`: globally available WordPress-shaped helpers for
  URLs, app data, i18n, body classes, email, hooks, sanitization, and runtime
  rendering.
- `app/includes/hooks.php`: lightweight priority-based action/filter registry.
- `app/includes/roles.php`: the only central role/capability map.
- `app/database/schema.sql`: baseline schema for a new installation.
- `app/database/schema-specs.php`: additive column/index/constraint metadata.
- `app/database/repair-specs.php`: opaque-ID normalization and orphan cleanup
  metadata shared by status and upgrade flows.
- `app/services/database/`: schema inspection, status, upgrade, repair, and DB
  context services.
- `app/services/install/`: install-state detection, config writing, installer
  screen/bootstrap/manager behavior, and locale selection.
- `app/services/update/`: remote manifest, release download/verification,
  filesystem replacement, maintenance lock, and update status.
- `app/services/geoip/`: MaxMind credentials, archive download/extraction,
  local database lookup, and status.
- `app/services/i18n/`: locale/catalog paths, loading, browser negotiation, and
  active locale/direction.
- `app/services/favicon.php`, `app/services/social-preview.php`: uploaded image
  validation/storage and generated public assets/metadata.
- `app/services/crypto.php`, `app/services/totp.php`, `app/utils/security.php`,
  `app/utils/secrets.php`: encryption/signing, 2FA, origin/cookie security, and
  token hashing.
- `app/services/mailer.php`, `app/services/notifications.php`: mail transport,
  templates, and transactional email.
- `app/templates/emails/`: plain/HTML default templates plus locale overrides.
- `app/bin/setup-database.php`: CLI database create/schema/bootstrap for Docker
  and local development.
- `app/bin/update-geoip.php`: CLI GeoLite2 refresh using saved credentials.

### Release front controller and installer

- `site/index.php`: installed-release front controller. It handles maintenance,
  install-state redirects, API pass-through, favicon/manifest files, public
  short-link requests, and dashboard HTML injection.
- `site/setup-config.php`: installer steps 0–2; validates DB settings and writes
  an initial `config.php`.
- `site/install.php`: final installer step; creates schema/owner, logs the owner
  in, then removes install-only values from `config.php`.
- `site/config-sample.php`: template for generated/base config values; do not
  edit as if it were active configuration.
- `site/database-connection-error.php`: safe recovery page when config exists
  but the DB cannot be reached.
- `site/content/index.php`, `site/content/plugins/index.php`, and upload
  placeholder files protect content directories from directory listing.

### React dashboard

- `ui/PeakURL.tsx`: async i18n initialization, generator tag, and React root.
- `ui/App.tsx`: Redux/theme/notifications providers plus `BrowserRouter`.
- `ui/router/index.tsx`: all client routes, route effects, admin-only routing,
  and redirects.
- `ui/pages/layout/`: public auth layout and protected dashboard shell/sidebar.
- `ui/pages/auth/`: login, forgot-password, and reset-password flows.
- `ui/pages/dashboard/`: dashboard overview, links, activity, users, tools,
  plugins, settings, and about pages.
- `ui/components/ui/`: reusable controls (buttons, inputs, modal, loading,
  notifications, select, dialog, skeletons, etc.).
- `ui/components/common/`: API error, branding, logo, preview image, and theme
  controls.
- `ui/components/providers/`: Redux, authentication initialization, theme, and
  notification contexts.
- `ui/store/slices/api/base.ts`: one RTK Query API instance with credentials
  included and shared tag types.
- `ui/store/slices/api/{analytics,urls,user,system,webhook}.ts`: injected RTK
  Query endpoint groups and cache invalidation.
- `ui/api/api.ts`: canonical relative API route builders and encoded dynamic
  parameters. Add API paths here rather than hard-coding them in components.
- `ui/api/types/`: API DTOs, request payloads, and response types.
- `ui/data.ts`: safe parsing and updating of PHP-injected `window.__PEAKURL__`.
- `ui/constants.ts`: runtime site/API URL, basename, host, version, locale data,
  and Vite fallbacks.
- `ui/i18n/`: `@wordpress/i18n` setup, language catalog, direction, and types.
- `ui/utils/`: shared URL/date/error/CSV/record/security/DOM helpers. The
  `ui/utils/index.ts` barrel is the preferred import surface.
- `ui/styles/`, page CSS, component CSS: the global/theme/layout styling system.

## Runtime request flow

### Local Docker development

1. Caddy serves `https://peakurl.dev` and proxies `/api/*` to `app:8000`, all
   other UI paths to `ui:5173`.
2. Vite serves the React source with `base: "./"`; it proxies API, public asset,
   and short-code fallback requests to the PHP app.
3. The PHP dev server uses `docker/app/router.php`. Existing files/assets are
   served directly; other paths are routed to `app/public/index.php`.
4. `docker/app/start.sh` installs Composer dependencies when the hash of
   `composer.json`/`composer.lock` changes, refreshes autoloading, runs
   `bin/setup-database.php`, and starts PHP on port 8000.

Main local endpoints:

- UI: `https://peakurl.dev` or `http://localhost:5173`
- API: `https://api.peakurl.dev` or `http://localhost:8000`
- Packaged installer: `https://peakurl.test`
- phpMyAdmin: `http://phpmyadmin.localhost`
- MySQL host port: `127.0.0.1:3307`

### Packaged release

1. `scripts/build-release.sh` builds the dashboard to `build/`, copies it as
   `app.html`, copies `site/` into the release root, copies production PHP into
   `release/peakurl/app`, includes language packs, and removes `.env`.
2. The deployed root `site/index.php` calls `InstallState::get_state()` before
   serving anything.
3. States are `needs_setup` (no config), `needs_install` (config but missing
   tables/site/owner), `database_connection_error`, and `ready`.
4. API requests while not ready receive a structured 503 recovery payload with
   installer/recovery URLs. Browser requests redirect to the matching installer
   page.
5. Ready dashboard paths (`/`, `/login`, `/forgot-password`,
   `/reset-password/*`, `/dashboard/*`) receive `app.html`. PHP injects a safe
   `<base>` element and `window.__PEAKURL__` before React boots.
6. Non-dashboard paths are sent to the PHP app, including public short-link
   redirects. Favicon aliases are served from generated content or fallback
   assets.

Do not remove the whole `release/` directory while `peakurl-test` is running;
the container bind-mounts it. Rebuild in place with `npm run release:build`.

## PHP API architecture

The API base is `/api/v1` (`Constants::API_BASE_PATH`). `Application` registers
the following route groups. All handlers return a JSON-ready array through the
common `success`/`error` envelope unless a handler intentionally returns a
download/HTML response.

- Core: `GET /health`, `GET /system/i18n`.
- Auth: register, email verification, login, 2FA login verification, logout,
  forgot/reset password, API-key CRUD, 2FA setup/verify/disable/backup codes,
  and session revocation.
- Users: list/create users, current-user GET/PUT, managed-user PUT/DELETE.
- URLs: list/export/show/create/bulk-create/update/delete/bulk-delete.
- Analytics: dashboard summary, recent activity/clicks, paginated history,
  activity deletion, per-link location and stats.
- Webhooks: list/create/delete.
- System: notices, general settings, system status, CAPTCHA, GeoIP, mail/test
  mail, update status/check/apply/reinstall/database upgrade.
- Public root routes (not under `/api/v1`): `/{id}`, `/{id}/`, for GET/HEAD/POST.
  These resolve a short code or alias and implement password, expiration,
  CAPTCHA, social-preview, click recording, and redirect behavior.

Route registration order matters when adding routes. API routes are prefixed by
`/api/v1`; public redirects are registered with an empty prefix. The router
only supports exact paths plus single-segment `{param}` placeholders and scans
routes in registration order for the requested HTTP method.

`Request::from_globals()` strips the script mount path, parses JSON and URL-
encoded bodies, preserves multipart `$_POST` and `$_FILES`, captures query,
cookies, headers/server data, and lets the router attach route params.

`Application::run()` validates browser-originating mutations against `SITE_URL`,
bootstraps/repairs the site schema, fires `admin_init` for API paths, dispatches
through the REST hooks, catches `ApiException`, and hides unexpected exception
details unless development/debug mode is enabled. `HEAD` returns headers/status
without emitting a body.

Important extensibility hooks include `init`, `admin_init`, `rest_api_init`,
`rest_pre_dispatch`, `rest_request_before_dispatch`, `rest_post_dispatch`,
`dashboard_data`, `site_url`, `api_base_url`, `site_name`, `body_class`,
`html_lang`, and `text_direction`. New PHP extensions should use these hooks or
route registration rather than modifying unrelated controller behavior.

## Store, authorization, and persistence

`Store` is intentionally the central facade. It owns the connection, DB wrapper,
settings/users/links APIs, roles, crypto, TOTP, CAPTCHA, favicon, social
preview, GeoIP, mail, notification, and i18n services. Public domain methods
are spread across traits:

- `BootstrapTrait`: schema repair, owner bootstrap, install welcome email.
- `AuthorizationTrait`: capability checks and owner/global link/analytics SQL
  scopes.
- `SessionsTrait`: session-cookie/API-key authentication, session pruning,
  session metadata, and revocation.
- `AccountsTrait` and `app/traits/accounts/*`: login, profile/user management,
  password/email flows, 2FA, API keys, backup codes, and validation.
- `SettingsTrait`: option access and general site settings.
- `LinksTrait`: URL CRUD, bulk import/delete, access checks, password/CAPTCHA,
  social preview, link listing/export, and click redirects.
- `AnalyticsTrait`/`AnalyticsSupportTrait`: aggregate stats, activity, recent
  clicks, date ranges, metric grouping, and click recording.
- `WebhooksTrait`: per-user webhook CRUD and one-time secret exposure.
- `SystemTrait`/`SystemSupportTrait`: system settings/status and GeoIP/update
  integration.
- `FormattingTrait`, `FindersTrait`, `HelpersTrait`: API shapes, lookups,
  sanitation, IDs, SQL helpers, JSON/date conversion, and common internals.

Roles are only `admin` and `editor`:

- Admins can manage users/site settings/mail/location/updates/webhooks/API keys,
  see and edit/delete all links, and see site-wide analytics.
- Editors can create and view/edit/delete links and analytics within the
  ownership rules, view site analytics as permitted by the capability map, and
  manage their own profile. They cannot manage users or site services.

Do not rely on UI guards for authorization. Every protected Store method must
enforce authentication/capabilities and, for editor access, owner scoping in
the SQL query or record check. The UI's `AdminOnlyRoute` and settings-tab guards
are only presentation/navigation safeguards.

Authentication checks API bearer keys first, then the signed/session cookie.
API keys are shown only at creation and stored hashed. Session tokens are
hashed in the DB and session cookies are HttpOnly with configured SameSite and
Secure behavior. Password-reset/email-verification lookup tokens are also
stored hashed. TOTP secrets and backup-code flows live in the users table; a
backup code is consumed atomically when used.

Use prepared values through `PeakURL_DB`; do not interpolate user values into
SQL. Dynamic identifiers/order clauses must go through the existing query
sanitizers/maps. IDs are opaque random values in current schema; upgrade repair
code normalizes older prefixed IDs in `clicks` and `webhooks`.

## Database and schema changes

The managed tables are `settings`, `users`, `api_keys`, `sessions`, `urls`,
`clicks`, `audit_logs`, and `webhooks`. Foreign keys use cascading deletes for
owned records; audit-log user/link references are nullable/cascading according
to the baseline schema and repair metadata.

The current target schema version is `6`. New-install structure belongs in
`app/database/schema.sql`; additive compatibility metadata belongs in
`app/database/schema-specs.php`; repair/cleanup behavior belongs in
`app/database/repair-specs.php`. The database service is designed to be
idempotent: runtime bootstrap calls repair, and explicit admin/CLI upgrade flows
record schema version/timestamps/errors in settings.

When changing schema:

1. Update the baseline SQL.
2. Update `SchemaSpecs` for existing installations (columns, indexes, and
   constraints as applicable).
3. Raise `Constants::DB_SCHEMA_VERSION` when the managed version changes.
4. Add/adjust status and repair behavior if the change is not purely additive.
5. Verify both a fresh DB and an older DB through `setup-database.php` and the
   admin database-upgrade path.

Settings are key/value rows. Dashboard-managed site settings generally belong
   in the DB; bootstrap credentials/DB/site URL/auth keys/runtime paths belong
   in config/env. Sensitive saved credentials are encrypted through `Crypto`.

## Frontend architecture and routes

The React app uses one Redux store and one RTK Query `baseApi` with
`credentials: "include"`. Domain slices inject endpoints and use tags to
invalidate related lists/analytics/settings. API paths must be added to
`ui/api/api.ts`, and DTOs to the nearest relevant `ui/api/types/*.ts` file.

`AuthInitializer` performs the initial session/connection/install-recovery
check. `ProtectedRoute` repeats the session check for protected dashboard
routes, redirects unauthenticated users to `/login`, and displays a connection
error page for non-auth failures. API error helpers in `ui/utils/errors.ts`
(`extractErrorMessage`, `getErrorMessage`, `getErrorStatus`) are the shared way
to interpret unknown RTK Query errors.

Client route shape:

- Public: `/login`, `/forgot-password`, `/reset-password/:token`.
- Protected dashboard: `/dashboard`, `/dashboard/activity`, `/dashboard/about`,
  `/dashboard/links`, `/dashboard/settings/:tab`.
- Tools: export is available to authenticated users; import and system status
  are admin-only.
- Admin-only pages: `/dashboard/users` and `/dashboard/plugins`.
- `/dashboard/team` redirects to users; invalid paths render `NotFoundPage`.
- `/` redirects to `/dashboard`; the router uses the PHP-injected basename for
  subdirectory installs.

The dashboard feature layout is deliberately local: page-specific components,
types, and CSS live under the page; shared controls live under
`ui/components`. Prefer the nearest `types.ts` for reusable feature types.
Use explicit `.tsx` component names, `import type` for type-only imports,
extensionless imports, and `@/utils` for shared utility imports.

TypeScript is strict (`strict`, `noUnusedLocals`, `noUnusedParameters`,
`noFallthroughCasesInSwitch`, `noUncheckedSideEffectImports`). Prefer
`unknown` plus narrowing over `any`; keep assertions narrow/local. Component
props, especially destructured props, should have named types. Keep API shapes
independent from presentation components.

UI upload endpoints switch from JSON to `FormData` when images/removal flags are
present. The common helper is `ui/store/slices/api/formData.ts`. Do not set a
manual multipart `Content-Type`; let `fetch` set its boundary.

The PHP-injected app data (`window.__PEAKURL__`) carries base path, API/site URL,
site name/version/debug, locale/language direction/timezone/time format,
favicon, and the translation catalog. Use `ui/data.ts` helpers to read/merge
it. After settings changes that affect shell data, update app data and/or
favicon through the existing helpers so the current page reflects the change.

## Internationalization

PHP uses the `peakurl` gettext text domain and loads catalogs from
`content/languages`. The dashboard uses `@wordpress/i18n` and the injected
catalog, with `/api/v1/system/i18n` as the runtime fallback. Source strings are
extracted from PHP/TSX by the i18n scripts. New user-facing strings should use
the existing translation helpers rather than hard-coded copy, then refresh the
POT/PO/MO/JSON artifacts as appropriate.

Locale also controls HTML `lang`, text direction (`ltr`/`rtl`), date/time
formatting, and dashboard timezone. Avoid assuming UTC in UI display code; DB
timestamps are stored/queried as UTC and converted for configured site/user
presentation where the existing helpers do so.

## Security and public-link behavior

- Mutating API requests with an Origin/Referer are rejected unless they match
  the configured site origin. CORS allows credentials only for the configured
  origin.
- Passwords use PHP password hashing. API keys and lookup tokens are hashed;
  encrypted integration credentials use `Crypto` derived from auth config.
- Uploaded favicon/social images are validated and stored below content paths;
  preserve path validation and never trust user-supplied filenames.
- Public link requests must resolve only active, non-expired records and then
  enforce password/CAPTCHA access before recording the click/redirect.
- Click analytics stores a visitor hash and request metadata; GeoIP lookup uses
  the local MaxMind GeoLite2 City database and gracefully handles unavailable
  or private IPs.
- Update application uses remote manifest/checksum validation, a filesystem
  lock, `.maintenance`, backup/restore paths, and is intended for packaged
  releases. Source checkouts should not be treated as update-install targets.

When changing auth, installer, public redirect, uploads, update, or CORS code,
test both browser and direct API behavior and avoid exposing secrets in API
responses/logs/errors.

## Configuration and filesystem paths

Runtime config precedence is runtime environment variables, `.env`, generated
`config.php`, then defaults. Canonical keys include `DB_HOST`, `DB_PORT`,
`DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_CHARSET`, `DB_PREFIX`,
`SITE_URL`, `PEAKURL_ENV`, `PEAKURL_DEBUG`, auth key/salt, content/GeoIP paths,
update manifest URL, and session cookie settings.

Source checkout runtime values are normally in `app/.env` and Docker Compose
environment variables. An installed release uses root `config.php`; the final
installer removes owner/workspace/install-only secrets from that file.

Default writable/runtime areas include:

- `content/uploads/geoip/GeoLite2-City.mmdb` (ignored binary GeoIP DB).
- `content/uploads/favicon/` (generated favicon and webmanifest).
- `content/uploads/social-preview/` and link preview storage as configured.
- `content/languages/` (compiled catalogs).
- `content/debug.log` when debug is enabled.
- release updater temporary/lock/maintenance paths under the configured app/
  content storage.

Do not commit credentials, generated config, `app/vendor`, `node_modules`,
build/release artifacts, uploaded images, GeoIP binaries, or debug logs.

## Commands and verification

Install JS dependencies with `npm ci` (CI uses `--legacy-peer-deps` where
needed). Install PHP dependencies with Composer in `app/`.

Common commands from the repository root:

```bash
./scripts/setup-local.sh                 # one-time macOS local HTTPS setup
docker compose up --build                # full local stack
docker compose down
docker compose logs -f
npm run build                             # tsc -b + Vite production build
npm run lint                              # web + PHP standards
npm run lint:web
npm run lint:php
npm run lint:php:syntax
npm run format:check
npm run format
npm run release:build
npm run smoke:auth
```

For a PHP-only change, run `npm run format:php`, `npm run lint:php`, and
`npm run lint:php:syntax`. For a UI-only change, run the web formatter/linter
and `npm run build`. For installer, auth, update, schema, or routing changes,
also exercise the relevant Docker or packaged-release flow.

Formatting conventions are Prettier tabs, width 4, double quotes for web code,
ES5 trailing commas, and two-space YAML. PHP follows WordPress Coding
Standards through `phpcs.xml`, with tabs and a four-space tab width. Generated
`content`, `build`, `release`, `app/vendor`, and lockfile paths are excluded from
normal Prettier/ESLint scopes as configured.

## Change guidance and pitfalls

- Read the corresponding route in `app/includes/application.php`, controller,
  Store trait, and UI RTK Query endpoint before changing an API contract.
- Keep controller response envelopes and frontend DTOs synchronized. A normal
  response is `{ success, message, data }`; some list methods place `items` and
  `meta` under `data`.
- Preserve API route encoding. Dynamic UI route builders call
  `encodeURIComponent`; do not concatenate raw IDs/tokens/usernames.
- Preserve path-prefix behavior. Both PHP `Request` and React `BrowserRouter`
  support subdirectory installs; test a non-root base when changing routing,
  HTML injection, assets, cookies, or API URLs.
- Keep route-specific paths ahead of catch-all/public paths where method/path
  matching could otherwise become ambiguous.
- Do not put authorization only in React. Editors must be scoped server-side.
- Do not put dashboard-managed settings in generated config unless the existing
  install flow explicitly requires bootstrap-time configuration.
- Do not manually delete release output while `peakurl-test` is active.
- Do not use destructive Git/filesystem commands to “clean” generated output;
  rebuild or remove only a precisely identified artifact when necessary.
- Existing TypeScript lint has a few intentionally relaxed migration rules
  (`no-unused-vars`, `no-empty`, `prefer-const`, React refresh/export, and the
  effect state rule). Do not broaden these exceptions for new code.
- There is a plugins page and plugin-related hooks, but the visible plugin
  marketplace/install surface is currently a preview/coming-soon area. Check
  existing product direction before adding a new SaaS-style admin surface.

## Useful starting points by task

- API endpoint: `app/includes/application.php` → `app/controllers/*` →
  `app/store.php`/`app/traits/*` → `app/database/*`; mirror with
  `ui/api/api.ts`, `ui/store/slices/api/*`, and page usage.
- Login/2FA/session: `app/controllers/auth.php`, `app/traits/sessions.php`,
  `app/traits/accounts/auth.php`, `app/traits/accounts/security.php`,
  `ui/pages/auth/login`, and security settings hooks.
- Short-link/public redirect: `app/controllers/urls.php`,
  `app/traits/links.php`, `app/api/links.php`, and the links page/stats drawer.
- Analytics: `app/traits/analytics.php`, `app/traits/analytics-support.php`,
  `app/controllers/analytics.php`, `ui/store/slices/api/analytics.ts`, and
  dashboard/links stats components.
- Installer/runtime: `site/index.php`, `site/setup-config.php`,
  `site/install.php`, `app/services/install/*`, `app/includes/runtime-config.php`.
- Schema/upgrade: `app/database/schema.sql`, `schema-specs.php`,
  `repair-specs.php`, `app/services/database/*`, `app/bin/setup-database.php`.
- Release/update: `scripts/build-release.sh`, `app/services/update/*`,
  `app/traits/system-support.php`, and the updates settings tab.
- i18n: `app/includes/functions.php`, `app/services/i18n/*`, `ui/i18n/*`,
  `scripts/i18n/*`, and `content/languages/*`.
