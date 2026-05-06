# PT Fuel Tracker — Context for Claude

> Paste this into a Claude.ai Project description (or the start of a new chat) so any Claude conversation has the context to discuss, debug, or design changes for the Fuel Tracker app.

## What it is

**PT Fuel Tracker** is the in-house fuel and receipt tracking app for **Plateau Trees** (an Australian arboriculture company). It's owned and operated by Jessica (jessica@plateautrees.com.au).

- **Drivers** use it on their phones to submit fuel receipts. The app uses Anthropic's Claude API to OCR the receipt photo *and* the fleet card photo in one shot, extract the relevant fields, and prompt the driver to confirm.
- **Admins** use it on desktop to reconcile entries against bank/fleet-card statements, flag suspicious entries, manage vehicles and services, and export reports.

## Where it lives

- **GitHub repo**: https://github.com/Maxip123-SURF/Plateau-Fuel-Tracker (private)
- **Hosting**: Vercel — auto-deploys from `main` branch on push. No `vercel.json`; uses Vite defaults from the repo root.
- **Backend**: Supabase project `gevlhzzlivsiyxaysskv.supabase.co` — database tables (`fuel_entries`, `app_settings`) and a `receipts` storage bucket for image blobs.
- **AI**: Anthropic Claude API (each user supplies their own API key in Settings — not stored in source).

## Tech stack

- **Frontend**: React 19, Vite 8, single-page app, no router (uses a `view` string state to switch between seven views)
- **Architecture quirk**: the entire app is one ~15,000-line file at `src/App.jsx`. Everything — components, helpers, the database client, OCR prompts, Excel export logic — lives in that one file. Splitting it is on the wishlist but not done yet.
- **Other deps**: `@supabase/supabase-js`, `xlsx` + `xlsx-js-style` (Excel exports), `@vercel/analytics`, `@vercel/speed-insights`
- **Storage layers**: `localStorage` is the fast local cache (with quota-eviction logic for receipt image blobs); Supabase is the source of truth; the app tolerates Supabase being temporarily unreachable.

## App.jsx structure (high-level line ranges)

| Lines | What's there |
|---|---|
| 1–271 | Supabase client + `db` helper (load/save/delete `fuel_entries`, `app_settings`, ensure `receipts` bucket) |
| 285–331 | `window.storage` localStorage shim with quota eviction |
| 334–413 | `DIVISIONS`, `VT_COLORS`, service intervals (km vs hours-based for Excavator/Stump Grinder/Mower/Landscape Tractor) |
| 413–730 | Driver/card lookups: `DRIVER_CARDS`, name aliases/nicknames, `lookupRego`/`lookupCardNumberByRego`, auto-reconcile drivers |
| 787–924 | Image utils (`fileToB64`, `compressImage`, EXIF date extraction) |
| 928–1191 | `buildReceiptScanPrompt` / `buildCardScanPrompt` (the Claude prompts) |
| 1192–2350 | Receipt normalization, learned-corrections layer, fuzzy fleet-card matching |
| 2351–2422 | `claudeScan` — the Claude API client + per-task model config (orientation/receipt/card) |
| 2423–2722 | Date/sort/export helpers (vehicle-type Excel, fleet-card monthly summary) |
| 2724–3853 | All sub-components: `Toast`, `Pill`, `PhotoUpload`, `ScanCard`, modals (`EditEntry`, `EditVehicle`, `Service`, `ManualEntry`, `ConfirmDialog`, `ReceiptViewer`) |
| 4023–14866 | The `App` component itself (~10,800 lines) — all 7 views: `submit` (driver wizard), `dashboard`, `data`, `drivers`, `cards`, `reconcile`, `settings` |

## The seven views

- **submit** — 4-step driver wizard (driver name + rego → photo → review → confirm). Mobile-optimised.
- **dashboard** — admin overview: spend, fleet table, worsening/overdue/approaching-service vehicles
- **data** — full entries table with search, filtering, edit, soft-delete (30-day trash)
- **drivers** — per-driver views and admin tools (delete driver, merge driver names)
- **cards** — fleet card monthly summary with inline header editing
- **reconcile** — side-by-side spreadsheet view of imported fleet-card CSV vs app entries; auto-create missing entries for non-app managers
- **settings** — API key, Claude model selection per task, admin passcode, learned corrections, learned-card mappings

## Things that would surprise a fresh reader

- **Two Vite apps in the repo**. The real one is at the repo root (`src/App.jsx`). There's also a `preview-app/` folder with its own smaller app — that's a sandbox, not deployed, can be ignored.
- **Two leftover JSX files at the repo root** (`fuel_app (1).jsx`, `fuel_app_2.jsx`) — old chat-export artifacts, not used by the build. Cleanup pending.
- **The Supabase anon key is hardcoded in `App.jsx`** at line ~16. That is intentional and safe — anon keys are designed to be public, and Row Level Security enforces actual access.
- **Driver/vehicle data lives in the source code** (`DRIVER_CARDS` and `REGO_DB` arrays inside App.jsx). There are two `.cjs` scripts in `scripts/` that text-mangle App.jsx to apply bulk fleet-card corrections — they break if those arrays are moved out of source.
- **No tests, no TypeScript, no path aliases.** ESLint is configured but minimal.
- **Service tracking distinguishes km-based vs hours-based** vehicles. Excavator, Stump Grinder, Mower, Landscape Tractor are tracked in hours (500hr service interval, warn at 450hr); everything else is km (10,000km / warn at 8,000km).
- **Multiple "learned" layers**: `learnedDB` (rego-to-vehicle metadata learned from submissions), `learnedCardMappings` (fleet card → rego corrections), `learnedCorrections` (station name fixes, fuel-type corrections, digit OCR patterns). All sync to Supabase `app_settings` and cache in localStorage.

## What I'd want you (Claude) to do

If a user asks you to help with the Fuel Tracker, you can:

- **Discuss** the app's behaviour, structure, or design tradeoffs
- **Read and explain** specific parts of `App.jsx` (if the user pastes in a section)
- **Suggest** code changes — but you can't push them yourself from a Claude.ai chat. The user (Jessica) has Claude Code set up on her PC at `C:\PT Automation OFFICIAL\App code\Plateau-Fuel-Tracker\` — *that* Claude can edit and push. From a Claude.ai chat, output the change as a diff or copy-pasteable snippet.
- **Design** new features or UI improvements before implementation
- **Debug** issues from error messages, screenshots, or code excerpts the user shares

## What you should NOT assume

- That you can read or modify the live database. Schema/RLS changes need Supabase dashboard or CLI access — neither the user's Claude.ai chat nor Claude Code currently has Supabase MCP installed.
- That the user remembers exact file paths or line numbers — ask them to paste relevant code if you need it.
- That a change discussed in chat will be deployed automatically — only Claude Code on Jessica's PC pushes commits. From a Claude.ai chat, the deliverable is a clear diff for Jessica or her Claude Code session to apply.
