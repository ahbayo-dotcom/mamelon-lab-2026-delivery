# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Mamelon Lab Courier (تطبيق المندوب)** — a single-file PWA used by delivery couriers (مندوبين) of a dental lab to pick up, deliver, and return case work-orders. It shares its Supabase database with the separate `mamelon-erp` project (the lab's desktop ERP) but is otherwise a fully independent codebase — **do not merge code between the two projects.**

There is no build step, no framework, and no package manager: `index.html` is the entire application (HTML + CSS + vanilla JS in one file), designed to be installed directly on courier phones (wrapped as an APK via a PWA→APK conversion service) or opened as a PWA.

## Commands

None. There is no build/lint/test tooling in this project — it's a static HTML file. Open it directly in a browser, or serve it via any static file server for local testing (e.g. `python -m http.server`) since `file://` origins disable `localStorage`, which the app depends on for session persistence.

## Architecture

Single file: `index.html` (~1,470 lines as of 25 Aug 2026). Structure:
- Inline `<style>` block (~lines 1-640): all CSS, custom properties for the navy/gold brand palette.
- Inline HTML body (~640-765): login screen, home screen (3 tabs: البحث/سلتي/مرجعة), sticky bars.
- Single `<script>` block (~765-end): all app logic, talks directly to Supabase's REST API (PostgREST) via `fetch` — no SDK, no backend of its own.

Also in this folder: `manifest.json` (PWA manifest), `icon.jpg` (app icon).

## Data / Backend

- **Database:** Supabase project `qhhqodxbhhvhkjtzmmsh` — the *same* project used by `mamelon-erp`. Any change to `cases`/`deliveries`/`returns` from either app is immediately visible to the other (no manual sync).
- **Tables used:** `cases` (read/update `status`, `current_holder`, `picked_up_at`), `deliveries` (insert on delivery), `returns` (insert on return), `delegate_directory` (a view, read-only, used for PIN auth).
- **Auth:** 4-digit PIN matched directly against `delegate_directory`, stored client-side in `localStorage` (`mamelon_delegate_user`) — UI-level only, not real Supabase Auth. Same trust model as `mamelon-erp`.
- **Notifications:** Telegram bot sends a message to one fixed group chat on every delivery/return. As of 25 Aug 2026 (ADR-C03) this goes through a Supabase Edge Function (`telegram-notify`) instead of calling `api.telegram.org` directly from the client — the bot token lives server-side only, as a Supabase secret. See `docs/06_API.md`.
- **QR/barcode scanning:** `html5-qrcode` library, loaded from `unpkg.com` CDN.

## Business Rules (confirmed directly by the user — do not re-litigate as bugs)

- **BR-C01:** The product-type list shown to couriers (`PMMA, ZIRCON, EMAX, Implant Zircon, Night Guard`) is a deliberately simplified fixed list — the real `products` catalog in Supabase has 13 items. This is intentional (keeps the courier's choice simple), not a sync bug. Don't propose wiring it to the `products` table unless explicitly asked.
- **BR-C02:** Confirming delivery/return operates at the `case_number` level, not per individual database row. A single `case_number` can have multiple rows (different materials — mirrors BR-009 in `mamelon-erp`), and marking one card "delivered" marks *all* rows sharing that case_number as delivered, while only one `delivery_stage` gets recorded in the `deliveries` log. **This is accepted, intended lab behavior** — the user does not track per-material delivery status and does not want this changed. Do not propose splitting delivery tracking per material.
- **BR-C03:** Product-type selection for a delivery is always manual, and intentionally independent from `cases.product_type` (which records "the final product" regardless of actual milling stage — e.g. PMMA is often just a trial mill, not the real delivered material). Never auto-fill or pre-select a delivery form's product-type buttons from `cases.product_type`.
- **BR-C04:** `confirmPickup()` already transitions a case to `status='out_for_delivery'` the moment it's added to the cart (not at delivery time) — this is correct, working behavior, not something to "fix."

## Known Issues (documented, not scheduled — do not silently fix without asking)

1. **✅ Telegram bot token exposure — fixed 25 Aug 2026 (ADR-C03).** The token no longer ships to the client at all; `sendTelegramNotification()` now calls a Supabase Edge Function (`telegram-notify`) which holds `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` as server-side secrets. **Operational step still pending (not a vulnerability, just setup):** the user must set those two secrets via the Supabase dashboard or CLI before notifications actually go out — until then the function returns `{"error":"Server not configured"}` and delivery/return confirmations still succeed, just silently without a Telegram ping.
2. **✅ Stored XSS — fixed and verified 25 Aug 2026 (ADR-C04).** An `escapeHtml()` helper is now applied to every DB-sourced value injected via `innerHTML` in `renderSearchedCases()`, the cart-rendering code inside `loadCart()`, and `loadReturned()` — including values used as HTML attributes (`data-case`, `data-product`, etc.), not just visible text. Verified live against production with an actual `<script>`/`<img onerror>` payload — rendered as literal text, no code execution.
3. **🟢 No offline support** (no IndexedDB queue, no Service Worker) — unlike `mamelon-erp` which has one (ADR-010/011 there). Possibly acceptable by design since live connectivity is needed anyway to safely check `current_holder=is.null` and avoid two couriers claiming the same case. Informational only — no action requested.

## Standing Rules

1. **This project is fully separate from `mamelon-erp`** despite sharing a Supabase database. Never merge code, copy files, or assume a React/Vite build exists here — this is raw HTML/vanilla JS by design.
2. **Don't propose a major restructure** (converting to React, splitting into multiple files) unless explicitly requested — staying a single file is intentional, for easy deployment to courier phones.
3. **Before any visible/behavioral code change:** describe the proposed change and get explicit confirmation first, same standing rule as `mamelon-erp`.
4. **After completing any edit, before ending the turn, update:**
   - `docs/09_PROGRESS.md` — a dated entry for what changed and why
   - `docs/12_AGENT_HANDOFF.md` — current state, completed-tasks list
   - `docs/10_DECISIONS.md` — only if the change is an architectural decision (new ADR)
   - `docs/07_BUSINESS_RULES.md` — only if a new BR-Cxxx rule is established

No task is considered done until it's documented in the same session.
