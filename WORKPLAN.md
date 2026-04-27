# Natoof Arabic Font Fix — Workplan

**Project:** natoof.ae — restore Arabic fonts on `/` (top menu) and `/ar/` (full Arabic site)
**Client:** Mariam
**Engineer:** Vladimir
**Date:** 2026-04-27
**Estimate:** ~1h 15m total work (Naskh kit already on server — §3 — saves the original 8 min budget for it; partly offset by the 4th `@import` discovered in `/ar/css/style.css`)

> ⛔ **READ FIRST — DO NOT CHANGE THE UI UNDER ANY CIRCUMSTANCES.**
> This is a font-restoration task only. No layout, color, spacing, markup, class-name, library, or "cleanup" changes are permitted. See **§0** for the full constraint, and **§7** for live task status (Completed / In Progress / Remaining / Decisions / Out-of-scope).

---

## 0. Hard constraint — DO NOT CHANGE THE UI

**The existing UI must not change under any circumstances.** This is a font-restoration task only. The goal is to bring the page back to **exactly** how it looked before Google retired the Early Access endpoint — nothing more.

Specifically, the following are **forbidden**:

- Renaming or removing existing CSS classes, IDs, or selectors.
- Changing colors, sizes, weights, line-heights, letter-spacing, margins, padding, or any layout values.
- Substituting a different font family (e.g. Noto Naskh / Noto Kufi) — the families must remain literally `Droid Arabic Naskh` and `Droid Arabic Kufi` so every existing `font-family:` reference resolves unchanged.
- Adding new HTML elements, wrappers, or rearranging markup.
- "Cleanup" of unrelated CSS, dead rules, vendor prefixes, or legacy `style-original.css`.
- Reformatting / reindenting files. Touch only the lines that need to change.
- Upgrading dependencies, replacing jQuery/Bootstrap/etc., or bumping any library version.
- Adding analytics, preconnect hints, or any new `<link>`/`<script>` tags beyond what's strictly required for the font fix.

Allowed changes are limited to:

1. Deleting the two dead `@import` lines that point at `fonts.googleapis.com/earlyaccess/...`.
2. Adding `@font-face` blocks for the self-hosted Droid Naskh + Droid Kufi files at the top of `css/style.css`.
3. Uploading the new `/font/Droid*` files.
4. (Only if DevTools proves it necessary) one MIME-type line in `.htaccess`.

**Verification rule:** after the fix, a screenshot of the page in its working state must be visually identical to the pre-bug version aside from the Arabic glyphs being the correct typeface again. If anything else shifts — spacing, weight, color, layout — stop and roll back.

---

## 1. Problem

The site loads Arabic via Google's **Early Access** font directory:

- `@import url(https://fonts.googleapis.com/earlyaccess/droidarabicnaskh.css)` (Naskh)
- `@import url(//fonts.googleapis.com/earlyaccess/droidarabickufi.css)` (Kufi)

### 1.1 Live-site diagnosis (verified in browser DevTools)

Confirmed against `https://natoof.ae/ar/` and `/ar/contact/` with Chrome DevTools open:

- The Arabic top-menu items and the hero `<h1 class="myarabicheading">` are rendering in a **system Arabic fallback** (Tahoma / Segoe UI Arabic on Windows, etc.) — not Droid Kufi. Letterforms are visibly plain compared to the original.
- DevTools → Elements → Styles **does** show an `@font-face` block declaring `font-family: 'Droid Arabic Kufi'` — pointing to `//fonts.gstatic.com/ea/droidar...` for `eot / woff2 / woff / truetype` sources. So the `earlyaccess/*.css` file itself still resolves and registers the family.
- The page's own selectors (e.g. `style.css:12575`, the inline `font-family: 'Droid Arabic Kufi'` rules) are still firing — but the inherited computed style on `<html>` falls through to `font-family: sans-serif`, because every `src:` URL inside that `@font-face` now 404s on Google's CDN.
- Result: family is registered → no usable bytes → browser silently falls back to `sans-serif` → system Arabic font.

#### 1.1.1 Network-tab evidence (captured on `/ar/`)

DevTools → Network → Font filter, hard reload, shows the `droidarabickufi.css` file itself returns **200** but every binary it references **404s with 0.0 kB**:

| File requested by `droidarabickufi.css` | Status | Size |
|---|---|---|
| `DroidKufi-Bold.woff2` | **404** | 0.0 kB |
| `DroidKufi-Regular.woff2` | **404** | 0.0 kB |
| `DroidKufi-Regular.woff` | **404** | 0.0 kB |
| `DroidKufi-Bold.woff` | **404** | 0.0 kB |
| `DroidKufi-Bold.ttf` | **404** | 0.0 kB |
| `DroidKufi-Regular.ttf` | **404** | 0.0 kB |

The same pattern is expected for `droidarabicnaskh.css` (loaded only by the English `/index.html`, not by `/ar/` — which is why no Naskh 404s appear on the Arabic homepage capture). Independent Open Sans request from Google Fonts CSS API succeeds (`memvYaGs126MiZpBA-...`, 200) — not in scope.

#### 1.1.2 Implications for the fix

- The breakage is **not** "the import is dead." It is "the binary font files behind the import are dead." We don't need to invent a new `font-family` name or rewrite any selector — we just need the `@font-face` `src:` URLs to point to local files instead of `fonts.gstatic.com/ea/`. Every existing `font-family:` reference (menu links, h1 quote, `style.css:12575`, inline styles) resolves unchanged the moment the bytes load from `natoof.ae/font/`.
- §0's "no UI changes" rule is achievable: the selectors and family names already match — we are only swapping where the bytes are served from.
- **Filenames to use locally:** mirror Google's exact names — `DroidKufi-Regular.woff2 / .woff / .ttf` and `DroidKufi-Bold.woff2 / .woff / .ttf`, plus the same shape for Naskh (`DroidNaskh-Regular.{woff2,woff,ttf}`, `DroidNaskh-Bold.{woff2,woff,ttf}`). Three formats per weight is sufficient for any browser still in use; `.eot` / `.svg` are obsolete (IE8 / iOS<5) and can be skipped to keep the kit small.
- Both **Droid Arabic Naskh** and **Droid Arabic Kufi** are referenced (Naskh on English homepage body, Kufi everywhere else) — both must be self-hosted.

## 2. Approach

Self-host both font families via `@font-face`. This restores the exact previous look (same family files, just delivered from `natoof.ae`) and removes the external dependency that just broke. Both fonts are Apache 2.0 licensed.

Alternative considered: switch to Noto Naskh Arabic / Noto Kufi Arabic (the modern Google replacements). Rejected because it would change visual weight/metrics and require re-tuning every Arabic block.

## 3. Current state of repo

- `c:\natoof-fix\backup\` — untouched FTP download. Now contains both trees:
  - `backup/` (root): `css/`, `font/`, `index.html`.
  - `backup/ar/`: `css/`, `font/`, `index.html`.
- `c:\natoof-fix\working\` — working copy mirroring the same shape. Edits happen here only.
- **Remote `/font/`** — Montserrat + FontAwesome + et-line. **No Droid files** on the server yet (still pending FTP upload — Step 5).
- **Remote `/ar/font/`** — Montserrat + FontAwesome + et-line **plus a Droid Arabic Naskh webfont kit already in place**: `DroidArabicNaskh_gdi.{eot,woff,ttf,svg}` (Regular) and `DroidArabicNaskh-Bold_gdi.{eot,woff,ttf,svg}`. Sample `.css` and `.html` ship with it. Matches the kit shape in client's `info/1.png`. **No `.woff2`** in this kit.
- **Local `working/font/` and `working/ar/font/`** — fully staged with all binaries the patched CSS references:
  - Naskh `_gdi` kit (Regular + Bold, `.eot/.woff/.ttf/.svg`) in **both** trees.
  - Kufi kit (Regular + Bold, `.woff2/.woff/.ttf/.eot`) in **both** trees, named `DroidKufi-{Regular,Bold}.{woff2,woff,ttf,eot}`.
- **Server still missing all Kufi files and the `_gdi` Naskh copy under `/font/`** — Step 5 will push these.

### 3.1 Family-name caveat on the existing Naskh kit

`ar/font/DroidArabicNaskh-Regular.css` declares `font-family: 'DroidArabicNaskh-Regular'` (single token), and the Bold sibling declares `'DroidArabicNaskh-Bold'`. The pages use `font-family: 'Droid Arabic Naskh'` (with spaces) and rely on `font-weight` to switch between regular and bold. So we **cannot** just `@import` the bundled `.css`. We will write our own `@font-face` blocks pointing at the same `_gdi` files but exposing them as one family `Droid Arabic Naskh` with `font-weight: 400` and `700`. The bundled `.css` / `.html` specimens stay on disk but go unreferenced.

## 4. Files to change

| File | Line | Current | Action |
|---|---|---|---|
| `working/index.html` | 34 | `@import .../earlyaccess/droidarabicnaskh.css` | **Delete** |
| `working/index.html` | 39 | `font-family: 'Droid Arabic Naskh', serif;` | Keep |
| `working/index.html` | 67 | inline `font-family: 'Droid Arabic Kufi', ...` (AR menu) | Keep |
| `working/index.html` | 86 | inline `font-family: 'Droid Arabic Kufi', ...` (h1 quote) | Keep |
| `working/ar/index.html` | 35 | `@import .../earlyaccess/droidarabickufi.css` | **Delete** |
| `working/ar/index.html` | 39 | `font-family: 'Droid Arabic Kufi', sans-serif;` | Keep |
| `working/css/style.css` | 27 | `@import .../earlyaccess/droidarabickufi.css` | **Replace with `@font-face` block (Naskh + Kufi)** |
| `working/css/style.css` | 12575 | `font-family: "Droid Arabic Kufi", ...` | Keep |
| `working/ar/css/style.css` | 27 | `@import .../earlyaccess/droidarabickufi.css` | **Replace with `@font-face` block (Kufi only — Arabic site doesn't use Naskh)** |
| `working/ar/css/style.css` | 139–153 | malformed `@font-face` blocks with no `src:` (zero-effect, predate the bug) | **Leave untouched** per §0 (don't "clean up" unrelated code) |
| `working/ar/css/style.css` | 4632–4676 | commented-out `@font-face` blocks for Open Sans (inside `/* … */`) | **Leave untouched** per §0 |
| `working/ar/css/style.css` | 140, 148, 163, 184, 1217, 4784, 4892 … 11037 | ~40 `font-family: "Droid Arabic Kufi"` rules | Keep — all resolve unchanged once the new `@font-face` is in place |

Plus on the FTP side:
- Upload **Droid Arabic Kufi** kit (Regular + Bold, `.woff2/.woff/.ttf`) to **both** `/font/` and `/ar/font/`.
- Upload a copy of the existing **Droid Arabic Naskh** `_gdi` kit from `/ar/font/` into `/font/` so the English homepage's `/css/style.css` (which uses `../font/...`) can resolve it from its own tree without crossing into `/ar/font/`.

### 4.1 Why edits land in two `style.css` files

The English homepage links `/css/style.css`; the Arabic site links `/ar/css/style.css`. They're separate files, each with its own dead `earlyaccess` `@import` at line 27. The same `@font-face` block must be added to both — but with different relative paths (`../font/...` resolves to `/font/` from the root tree and to `/ar/font/` from the Arabic tree).

## 5. Step-by-step

### Step 1 — Acquire font kits (~8 min)

**Naskh — already done.** The kit exists on the server at `/ar/font/DroidArabicNaskh*_gdi.{eot,woff,ttf,svg}` and is mirrored locally at `working/ar/font/`. We will reference these `_gdi` files directly via our own `@font-face` blocks (see §3.1 about the family-name mismatch). No `.woff2` is bundled — acceptable, the `.woff` already covers every browser the site supports.

**Kufi — still needed.** Acquire a Droid Arabic Kufi webfont kit (Regular + Bold):

- Source TTFs from the `googlefonts/droid` GitHub mirror or the Google Fonts archive.
- Run each TTF through the **Fontsquirrel Webfont Generator** (Optimal preset) — produces `.woff2/.woff/.ttf` plus a sample CSS.
- Save with these exact names (matches §1.1.1 and avoids invalid spaces in URLs): `DroidKufi-Regular.{woff2,woff,ttf}` and `DroidKufi-Bold.{woff2,woff,ttf}`.
- Place output in **both** `c:\natoof-fix\working\font\` and `c:\natoof-fix\working\ar\font\` so each tree's `style.css` can resolve `../font/Droid...` locally.

### Step 2 — Add `@font-face` rules in both `style.css` files (~15 min)

**Two edits — same shape, different scope.** Each tree's `style.css` is patched in isolation. Both edits replace **line 27** (the dead `@import`) with the appropriate `@font-face` block.

#### 2a. `working/css/style.css:27` — English homepage (needs Naskh + Kufi)

```css
/* === Arabic webfonts (self-hosted, replaces deprecated Google earlyaccess) ===
   Naskh `_gdi` kit comes from the existing /font/ upload (mirrored from /ar/font/).
   Kufi `.woff2/.woff/.ttf` are newly added in /font/. See §3.1 for the family-name
   override on Naskh — we expose the bundled DroidArabicNaskh-Regular / -Bold files
   under the family name 'Droid Arabic Naskh' that the page actually uses.        */
@font-face {
  font-family: 'Droid Arabic Naskh';
  src: url('../font/DroidArabicNaskh_gdi.woff') format('woff'),
       url('../font/DroidArabicNaskh_gdi.ttf')  format('truetype');
  font-weight: 400; font-style: normal; font-display: swap;
}
@font-face {
  font-family: 'Droid Arabic Naskh';
  src: url('../font/DroidArabicNaskh-Bold_gdi.woff') format('woff'),
       url('../font/DroidArabicNaskh-Bold_gdi.ttf')  format('truetype');
  font-weight: 700; font-style: normal; font-display: swap;
}
@font-face {
  font-family: 'Droid Arabic Kufi';
  src: url('../font/DroidKufi-Regular.woff2') format('woff2'),
       url('../font/DroidKufi-Regular.woff')  format('woff'),
       url('../font/DroidKufi-Regular.ttf')   format('truetype');
  font-weight: 400; font-style: normal; font-display: swap;
}
@font-face {
  font-family: 'Droid Arabic Kufi';
  src: url('../font/DroidKufi-Bold.woff2') format('woff2'),
       url('../font/DroidKufi-Bold.woff')  format('woff'),
       url('../font/DroidKufi-Bold.ttf')   format('truetype');
  font-weight: 700; font-style: normal; font-display: swap;
}
```

#### 2b. `working/ar/css/style.css:27` — Arabic site (Kufi only)

The Arabic templates do not use Naskh anywhere (grep confirms: only `Droid Arabic Kufi` references in the `/ar/` tree). Adding Naskh `@font-face` here would be unused weight — keep it lean.

```css
/* === Arabic Kufi webfont (self-hosted, replaces deprecated Google earlyaccess) === */
@font-face {
  font-family: 'Droid Arabic Kufi';
  src: url('../font/DroidKufi-Regular.woff2') format('woff2'),
       url('../font/DroidKufi-Regular.woff')  format('woff'),
       url('../font/DroidKufi-Regular.ttf')   format('truetype');
  font-weight: 400; font-style: normal; font-display: swap;
}
@font-face {
  font-family: 'Droid Arabic Kufi';
  src: url('../font/DroidKufi-Bold.woff2') format('woff2'),
       url('../font/DroidKufi-Bold.woff')  format('woff'),
       url('../font/DroidKufi-Bold.ttf')   format('truetype');
  font-weight: 700; font-style: normal; font-display: swap;
}
```

#### Notes for both edits
- `.eot` (IE) and `.svg` (old iOS) are intentionally omitted per §1.1.2.
- The `../font/` relative path matches the existing pattern used by `et-line` in `font.css:1` (root tree) and the equivalent in the `/ar/` tree.
- Existing references to `'Droid Arabic Kufi'` (`style.css:12575`, the ~40 lines in `ar/css/style.css`, plus inline `font-family:` rules in HTML) all resolve unchanged the moment these blocks are in place.
- The malformed `@font-face` at `ar/css/style.css:139–153` and the commented block at `4632–4676` are **left untouched** per §0 (no unrelated cleanup, even of broken code, on this engagement).

### Step 3 — Clean the two HTML files (~5 min)

**`working/index.html`:** delete line 34 (dead `@import`). Leave lines 39, 67, 86.

**`working/ar/index.html`:** delete line 35 (dead `@import`). Leave the rest.

Confirm `ar/index.html` already includes `../css/style.css` (it does — verify during the edit) so the `@font-face` rules reach it.

### Step 4 — Local pre-flight (~10 min)
- Open `working/index.html` in a browser via `file://` — verify the Arabic menu link "باللغة العربية" renders in Kufi.
- Open `working/ar/index.html` — verify body Arabic text renders in Kufi.
- DevTools → Network → Font: all requests 200, no 404.

### Step 5 — Upload to FTP (~12 min)
Order matters so users never see a half-broken state. Upload **all font binaries first**, then the CSS that references them, then the HTML that no longer needs the dead `@import`:

1. **Fonts (additive — never overwrites existing files):**
   - `/font/` ← copy of `DroidArabicNaskh{,-Bold}_gdi.{eot,woff,ttf,svg}` from `working/ar/font/`, plus the new `DroidKufi-{Regular,Bold}.{woff2,woff,ttf}`.
   - `/ar/font/` ← only the new `DroidKufi-{Regular,Bold}.{woff2,woff,ttf}` (Naskh kit is already there on the server).
2. **CSS:**
   - `/css/style.css`
   - `/ar/css/style.css`
3. **HTML:**
   - `/index.html`
   - `/ar/index.html`

### Step 6 — Server-side sanity (~5 min)
Check `.htaccess` (1,975 bytes, visible in FileZilla). If `woff2` MIME isn't configured, append:

```
AddType application/font-woff2 .woff2
AddType application/font-woff  .woff
AddType application/x-font-ttf .ttf
AddType application/vnd.ms-fontobject .eot
```

Only edit if DevTools shows a font being served as `text/plain` or `application/octet-stream`.

### Step 7 — Live verification (~15 min)
- Hard reload (Ctrl+F5): `https://natoof.ae/` and `https://natoof.ae/ar/`.
- DevTools → Network → Font filter → all Droid files return 200 from `natoof.ae`. **No remaining requests to `fonts.gstatic.com/ea/*` or `fonts.googleapis.com/earlyaccess/*`** (these are exactly the URLs that were 404ing before the fix — see §1.1).
- DevTools → Elements → Styles → for the AR menu link `a.header-links` and the hero `<h1 class="myarabicheading">`, the active `@font-face` block must show `src: url('../font/Droid...')` from `natoof.ae`, not `fonts.gstatic.com/ea/`.
- Computed style on the same elements resolves to `Droid Arabic Kufi` (and `Droid Arabic Naskh` where applicable) — not `sans-serif` fallback.
- Visual diff: place pre-fix screenshots (the `gstatic.com/ea/` 404 state captured in §1.1) next to post-fix and confirm only the Arabic glyph shapes changed — no shifts in spacing, weight, color, or layout (per §0).
- Cross-browser: Chrome, Firefox, Safari, mobile.
- Purge any CDN/Cloudflare cache if applicable.

### Step 8 — Hand-off
Send Mariam:
- Before/after screenshots.
- List of changed/added files.
- Note that `/backup/` (server-side and local) holds the originals for one-click rollback.

## 6. Edge cases & risks

- **MIME types:** if `.woff2` returns wrong content-type, browsers reject it. Mitigation in Step 6.
- **CORS:** not an issue — fonts are same-origin.
- **CDN cache:** if site sits behind Cloudflare, must purge after upload.
- **Family naming:** if a webfont generator renames the family, override with the exact `font-family:` string the page expects (`Droid Arabic Naskh` / `Droid Arabic Kufi`).
- **Rollback:** all originals live in `c:\natoof-fix\backup\` and were never modified on disk; reverting = re-upload the three files.

## 7. Task status

Legend: ✅ Completed · 🟡 In Progress · ⬜ Remaining · ⚠️ Blocker / decision needed

### ✅ Completed
| # | Task | Notes |
|---|---|---|
| 0.1 | Read client brief, FTP credentials, screenshots (`info/`) | All five files reviewed |
| 0.2 | Connected to FTP `server265.com:21` (FileZilla, `check / upwork026`) | TLS connection confirmed in screenshot |
| 0.3 | Downloaded full backup of `ar/`, `css/`, `font/`, `index.html` to `c:\natoof-fix\backup\` | Untouched — rollback source |
| 0.4 | Created working copy at `c:\natoof-fix\working\` (`ar/`, `css/`, `index.html`) | Edits will happen here only |
| 0.5 | Audited all references to broken Google `earlyaccess` imports | 3 files, 8 lines identified — see §4 |
| 0.6 | Confirmed remote `/font/` has NO Droid files | Naskh + Kufi kits must be uploaded |
| 0.7 | Workplan drafted and saved to `WORKPLAN.md` | This document |
| 0.8 | Diagnosed live site in browser DevTools (`/ar/`, `/ar/contact/`) | Confirmed root cause: `@font-face` registers correctly, but every `src:` URL on `fonts.gstatic.com/ea/` 404s → fallback to `sans-serif`. See §1.1 |
| 0.9 | Captured pre-fix Network log on `/ar/` showing 6 × `DroidKufi-*.{woff2,woff,ttf}` returning **404 / 0.0 kB** | Empirical "before" artifact for hand-off pack; concrete filenames extracted (see §1.1.1). |
| 0.10 | Mirrored remaining server tree to local: `backup/ar/css/`, `backup/ar/font/`, plus working copies | Surfaced 4th broken `@import` at `ar/css/style.css:27`, and revealed Naskh kit already present at `/ar/font/` — see §3, §3.1, §4 |
| 1 | Step 1 — Naskh kit | Already present at `/ar/font/`; mirrored to `working/ar/font/` | n/a |
| 2 | Step 1 — Kufi kit | Generated via Fontsquirrel; placed `DroidKufi-{Regular,Bold}.{woff2,woff,ttf,eot}` in `working/font/` and `working/ar/font/` (8 files × 2 trees) | done |
| 2.1 | Step 5 prep | Mirrored Naskh `_gdi` kit (Regular+Bold, all 4 formats) from `working/ar/font/` → `working/font/` so the English homepage can resolve it locally and the same files can be uploaded to `/font/` in Step 5 | done |
| 3a | Step 2a | `working/css/style.css:27` — `@import` replaced with Naskh + Kufi `@font-face` block | done |
| 3b | Step 2b | `working/ar/css/style.css:27` — `@import` replaced with Kufi-only `@font-face` block | done |
| 4 | Step 3 | `working/index.html:34` dead `@import` deleted | done |
| 5 | Step 3 | `working/ar/index.html:35` dead `@import` deleted | done |
| 6 | Step 3 | Verified `index.html:31` → `css/style.css` and `ar/index.html:33` → `/ar/css/style.css` (both link the patched stylesheets) | done |
| 6.1 | sanity | Verified all 16 font URLs referenced by both patched CSS files exist on disk; zero live `earlyaccess` / `gstatic.com/ea/` references remain in any working file | done |

### 🟡 In Progress
| # | Task | Notes |
|---|---|---|
| 7 | Step 4 — Local `file://` preview | **Awaiting your browser test.** Open `working/index.html` and `working/ar/index.html` in Chrome with DevTools → Network → Font filter + Disable cache. Goals: Arabic renders in correct typeface (Naskh on `index.html` body, Kufi on AR menu link / hero / `/ar/`), zero 404s, zero requests to `fonts.gstatic.com/ea/*` or `googleapis.com/earlyaccess/*`. Sandbox cannot drive a browser — needs your hands. |

### ⬜ Remaining
| # | Step | Task | Est. | Blocked on |
|---|---|---|---|---|
| 8 | Step 5 | FTP upload — `/font/` (Naskh `_gdi` copy + new Kufi files) and `/ar/font/` (Kufi files only) | 5 min | Task 7 pass |
| 9 | Step 5 | FTP upload `/css/style.css` and `/ar/css/style.css` | 2 min | Task 8 |
| 10 | Step 5 | FTP upload `/index.html` and `/ar/index.html` | 2 min | Task 9 |
| 11 | Step 6 | Inspect `.htaccess`; add font MIME types **only if** DevTools shows wrong content-type | 5 min | Task 12 evidence |
| 12 | Step 7 | Live verify on `https://natoof.ae/`, `/ar/`, `/ar/contact/` — Network tab, Computed styles, cross-browser | 15 min | Task 10 |
| 13 | Step 7 | Purge CDN/Cloudflare cache if applicable | 2 min | Task 12 |
| 14 | Step 8 | Hand-off to Mariam — before/after screenshots, file list, rollback note | 5 min | Task 12 |

### ⚠️ Decisions / open questions
- ~~**Filename convention for the kits.**~~ **Resolved (§1.1.1):** mirror Google's exact filenames — `DroidKufi-{Regular,Bold}.{woff2,woff,ttf}` and the existing `DroidArabicNaskh{,-Bold}_gdi.*` from the on-server kit.
- **CDN.** Unknown whether `natoof.ae` sits behind Cloudflare. Confirm before Step 7 so we know whether a purge is required.
- **`.htaccess` edits.** Do not pre-emptively modify; only act if Step 7 surfaces a MIME-type problem.
- **`font-display: swap`.** All 6 new `@font-face` blocks declare `font-display: swap`. Original Google `earlyaccess` CSS had no `font-display` (i.e. browser default = `block`). Strictly per §0 the conservative choice is to remove `font-display: swap`. Pending your call before Step 5.

### 🚫 Out of scope (explicitly NOT doing)
Per §0: no UI changes, no CSS cleanup, no markup edits, no library upgrades, no font substitution, no reformatting. Anything outside the four allowed change categories is rejected.

---

## 8. Done criteria

- [ ] `https://natoof.ae/` top-menu Arabic link renders in Droid Arabic Kufi.
- [ ] `https://natoof.ae/ar/` body Arabic renders in Droid Arabic Kufi.
- [ ] All 6 Droid Kufi files served as **200** from `natoof.ae/font/` (the same 6 that 404'd in §1.1.1): `DroidKufi-Regular.woff2`, `DroidKufi-Regular.woff`, `DroidKufi-Regular.ttf`, `DroidKufi-Bold.woff2`, `DroidKufi-Bold.woff`, `DroidKufi-Bold.ttf`.
- [ ] All 6 Droid Naskh files served as **200** from `natoof.ae/font/` when `/index.html` is loaded (same shape as Kufi).
- [ ] No external requests to `fonts.googleapis.com/earlyaccess/*` **or** `fonts.gstatic.com/ea/*` (the two endpoints proven dead in §1.1).
- [ ] Active `@font-face` rule in DevTools shows `src: url('../font/Droid...')` from `natoof.ae`, not `gstatic.com/ea/`.
- [ ] Computed `font-family` on the AR menu link and hero `<h1 class="myarabicheading">` resolves to `Droid Arabic Kufi`, not the `sans-serif` fallback observed pre-fix.
- [ ] Visual parity with the pre-breakage look (verify against client's reference if available).
