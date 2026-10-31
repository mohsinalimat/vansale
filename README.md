# Van Sale

Van sales app for field teams on Frappe v15 + ERPNext. One Vue 3 source tree ships as an installable web PWA and a signed Android APK (Capacitor), backed by a Frappe app that owns its DocTypes and API. Work done offline is queued on the device and sent in a fixed order when the connection returns.

## What it does

### Driver app

- **Sales:** draft, edit and submit invoices, with customer-specific prices, line and additional discounts, UOM change, tax-inclusive templates and cash-sale posting.
- **Collections:** payments against outstanding invoices, split across payment modes, or as advances.
- **Returns:** against an invoice or without one, with credit notes refundable later.
- **Customers:** create and edit customers, addresses and contacts, with KSA VAT validation and statements as a screen or PDF.
- **Route:** today's visit list, with start, end and skip, and optional GPS capture.
- **Van stock:** stock on hand and transfers into the van.
- **Printing:** invoices and receipts through configurable print formats. The APK uses Android's system print dialog.
- **Sync:** a Sync Errors screen where a failed queue entry can be edited or retried.
- **General:** dashboard, English and Arabic (RTL), phone and tablet layouts, PIN unlock.

### Office, in ERPNext

- **Setup:** `Vansale Settings` (global) and `Vansale Configuration` (per van). Settings resolve user → van → global.
- **Pages:** Van Driver Roster and Van Customer Assignment.
- **Reports:** Van Daily Settlement, Van Collection Summary, Van Stock Position and Van Visit Compliance.
- **Also included:** a Van Sales dashboard, per-van document numbering, and tax invoice and payment receipt print formats.

`frontend/src/features/van/printer.ts` (Bluetooth receipts), `scanner.ts` (barcode), `SignaturePad.vue` and `PhotoSlot.vue` exist, but no screen uses them yet.

## Screenshots

Demo data, captured at v1.0.29, so some details have since changed. The [user guide](docs/userguide/index.html) has the full set.

| Home | Invoice | Sales return | Van stock |
|---|---|---|---|
| <img src="docs/userguide/img/shot-03.png" width="200" alt="Home screen with today's sales, quick actions and recent activity"> | <img src="docs/userguide/img/shot-05.png" width="200" alt="Invoice detail with collect, print and return actions"> | <img src="docs/userguide/img/shot-07.png" width="200" alt="Sales return against an invoice, with return quantity and reason"> | <img src="docs/userguide/img/shot-08.png" width="200" alt="Van stock list with low and out-of-stock counts"> |

| Van Sales dashboard | Van Daily Settlement report |
|---|---|
| <img src="docs/userguide/img/shot-17.png" width="410" alt="Van Sales dashboard in ERPNext with number cards and charts"> | <img src="docs/userguide/img/shot-14.png" width="410" alt="Van Daily Settlement report showing sales, returns and cash collected per van"> |

## Structure

```text
vansale/              Frappe app: DocTypes, whitelisted API (api/), hooks, reports, print formats
frontend/             Vue 3 SPA, the single source for the web PWA and the APK
android-capacitor/    Capacitor shell; src/ is a symlink to ../frontend/src
scripts/              build, version bump, customer assets, docs generators
customers/            per-customer .env files and artwork
docs/                 technical docs, user guide, generated reference
docs-site/            static demo landing page
package.json          bench build hook that builds frontend/
```

Never add a second copy of a view under `android-capacitor/`. Editing `frontend/src` updates both targets.

## Install

Requires Frappe v15 with ERPNext, Python 3.10+, and Node with npm registry access on the server.

```bash
bench get-app https://github.com/EnfonoTech/vansale --branch develop
bench --site <site> install-app vansale
bench build --app vansale
/home/<bench-user>/frappe-bench/env/bin/pip install -e /home/<bench-user>/frappe-bench/apps/vansale
sudo supervisorctl signal QUIT frappe-bench-web:frappe-bench-frappe-web
```

Open `/vansale` on the site.

`bench build` runs the root `package.json` build, which compiles the SPA into `vansale/public/spa/` (gitignored). `bench migrate` re-provisions roles, permissions, custom fields, the dashboard, per-van series and the Capacitor CORS origins, and is safe to re-run.

The web app's name and icon come from `Vansale Settings`. Its service worker exists so the app can be installed and does no caching: opening the web app with no connection shows a "No connection" page, so only the APK can be started offline.

To upgrade, roll back or verify a deploy, see [`docs/OPERATIONS.md`](docs/OPERATIONS.md).

## Development

```bash
cd frontend
pnpm install
pnpm dev          # http://localhost:8082, proxies /api and /assets to 127.0.0.1:8000
pnpm test         # Vitest
pnpm type-check   # vue-tsc
```

Server tests:

```bash
bench --site <site> run-tests --app vansale
```

Regenerate the reference docs in the same commit as any change to a whitelisted endpoint, a DocType or `hooks.py`:

```bash
python3 scripts/gen-docs.py
```

Three conventions to know before changing code ([`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) explains why):

- **Writes are idempotent.** Each mutation goes through a purpose-built endpoint in `vansale/api/`, carrying a client-generated `client_id` that is checked against `Vansale Outbox`. Don't use `frappe.client.submit`.
- **The online path is not queued.** A mutation tries the network first, and only a network error falls through to the queue. A server rejection is shown to the driver straight away.
- **Drain order is fixed:** customers → stock entries → invoices → payments → returns → visits, so a queued document can rely on what it references.

## Release an APK for a new client

Needs JDK 21 to 25 (Android Studio's bundled JDK works), Android SDK platform 36, Node 22+ and pnpm on the build machine, with `JAVA_HOME` and `ANDROID_HOME` set. Run `pnpm install` in both `frontend/` and `android-capacitor/` first.

```bash
# 1. Config: copy the template, fill in app id, title, colours, site URL
cp customers/.env.example customers/.env.<client>

# 2. Artwork: add a 1024x1024 icon.png (logo.png and splash.png are optional)
mkdir -p customers/assets/<client>

# 3. Signing key: ONCE per client. Back the keystore up twice.
bash scripts/generate-keystore.sh <client>

# 4. Build: bumps the version, builds web + native, signs the APK
bash scripts/build-customer.sh <client>
```

Output lands in `dist/`: `vansale-<client>-<version>.apk`, plus `vansale-<client>-pwa.tar.gz`, a branded web build for that client's server. If no keystore password is found, the script builds only the tarball and skips the APK.

Rules that matter:

- **`CUSTOMER_APP_ID` must be unique per client** (`com.enfono.vansale.<client>`). Reusing an id makes the new APK overwrite the other client's app, and a different signing key on the same id makes Android refuse to install.
- **Never lose the keystore.** Re-signing with a new key forces every user to uninstall, which discards their offline queue.
- **Commit before building, and commit the bump after.** The script refuses a dirty tree and leaves the version bump uncommitted (`VANSALE_ALLOW_DIRTY=1` overrides this for local experiments).
- **Never hand-edit versions.** `scripts/bump-version.mjs` keeps `NATIVE_VERSION` and `versionCode` in lockstep. If they desync, Android silently skips the reinstall and the tester keeps running the old build.
- **`VITE_API_BASE` is optional.** Set it for an APK that connects straight to the client's site. Without it, the build warns and the app asks for the server address on first run.

See [`customers/assets/README.md`](customers/assets/README.md) for icon specs.

## Documentation

| | |
|---|---|
| [`docs/`](docs/README.md) | Documentation index |
| [`docs/userguide/`](docs/userguide/index.html) | End-user guide for drivers and office |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | How it fits together, and why |
| [`docs/OPERATIONS.md`](docs/OPERATIONS.md) | Install, upgrade, release, roll back |
| [`docs/SECURITY.md`](docs/SECURITY.md) | Roles, public surface, secrets |
| [`docs/reference/`](docs/reference/) | Generated: endpoints, DocTypes, hooks |
| [`docs/TESTING_CHECKLIST.md`](docs/TESTING_CHECKLIST.md) | Manual test checklist |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Roadmap |
| [`docs/AGENT_HANDOFF.md`](docs/AGENT_HANDOFF.md) | Project handoff |
| [`docs/LESSONS_LEARNED.md`](docs/LESSONS_LEARNED.md) | Lessons diary |
| [`docs/superpowers/plans/`](docs/superpowers/plans/) | Original phase plans |
| [`frontend/DESIGN.md`](frontend/DESIGN.md) | Frontend design reference |

`python3 scripts/build-docs-site.py` renders `docs/` into a static site at `dist/docs-site/`.

**Architecture pattern:** `~/.claude/skills/frappe-vue-pwa/SKILL.md`.
**Deploy + infra reference:** `~/.claude/skills/enfono-servers/SKILL.md`.

## License

MIT
