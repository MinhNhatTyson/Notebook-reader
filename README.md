# 📖 Notebook Reader

A personal, self-hosted reader for long `.txt` novels. Drop in a huge raw text file (10 MB+ is fine) and it becomes a comfortable reading experience with chapters, themes, fonts and a remembered position. The text itself is never modified; only how it is displayed changes.

> **Personal project.** Built for a single user (me), deployed on my own free Cloudflare account. The code is public for reference, but there is no public instance and no accounts.

## Features

- **Upload `.txt` books**, including multiple files at once. Parsing runs in a Web Worker so the UI stays responsive.
- **Encoding detection**: UTF-8, UTF-16 (LE/BE) and Windows-1258 (Vietnamese legacy).
- **Automatic chapter detection** (`Chương`, `Chapter`, `Hồi`, `Quyển`, `Tập`, `Phần`, `Volume`), with a fixed-size "Part N" fallback when no headings are found.
- **Reader settings**: 3 themes (light, sepia, dark), 6 fonts, font size, line height, text width, paragraph spacing, justify, first-line indent.
- **Two reading modes**: continuous scroll, or paged with swipe, tap zones and keyboard arrows.
- **Chapter drawer** with search.
- **Immersive reading**: auto-hiding header and a floating dock for chapters, theme and settings.
- **Library**: cover cards, progress bars, search, sort, delete.
- **Cloud sync** across devices: library, chapters and reading position (newest position wins).
- **PWA**: installable on a phone, with an offline app shell.
- **Mobile-first layout** that also works on desktop.

## Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 19, TypeScript, Vite, react-router |
| Local storage | IndexedDB via Dexie |
| Backend | Cloudflare Worker (`worker/index.ts`) |
| Cloud data | Cloudflare D1 (book info, chapters, progress) and KV (gzipped book text) |
| Hosting | Cloudflare Workers with static assets |

## How it works

```
Phone / Laptop (React app + IndexedDB cache)
        │  /api/*  (Bearer token)
        ▼
Cloudflare Worker
   ├─ D1: books, chapters, reading position
   └─ KV: book text (gzip)
Static files: dist/ served by Workers assets
```

Books are parsed in the browser, stored locally in IndexedDB, then uploaded in the background. On another device, a book appears in the Library as cloud-only (☁) and its text is downloaded the first time you open it.

## Getting started

### Prerequisites

- Node.js 20.19+ or 22.12+
- A free [Cloudflare account](https://dash.cloudflare.com/sign-up)

### Run locally

```powershell
npm install
npm run dev
```

In dev mode, Vite proxies `/api` to the deployed Worker (see `vite.config.ts`). Change the `target` there to your own Worker URL. Changes to `worker/index.ts` only take effect after a deploy.

### Set up your own Cloudflare backend

1. **Log in:**

   ```powershell
   npx wrangler login
   ```

2. **Create the D1 database and KV namespace:**

   ```powershell
   npx wrangler d1 create notebook-reader
   npx wrangler kv namespace create BOOKS
   ```

3. **Put the returned IDs into `wrangler.jsonc`** (`database_id` and the KV `id`).

4. **Create the table:**

   ```powershell
   npx wrangler d1 execute notebook-reader --remote --file=worker/schema.sql
   ```

5. **Set the access token** as a secret. Use a long random value, and never commit it:

   ```powershell
   npx wrangler secret put API_TOKEN
   ```

6. **Build and deploy:**

   ```powershell
   npm run build
   npx wrangler deploy
   ```

   Alternatively, connect the repo in the Cloudflare dashboard (Workers Builds) so every `git push` deploys automatically.

### Connect a device

Open `https://<your-worker>.workers.dev/sync`, paste the token and press **Save & sync now**. Use **Copy phone link** to open the same page on your phone with the token filled in.

> Use the `workers.dev` address on every device. `localhost` and `workers.dev` have separate browser storage, so books added on one will not appear on the other until synced.

## Project structure

```
src/
  components/   Header, ReaderDock, ChapterDrawer, SettingsPanel, ScrollView, PagedView, ChapterText
  pages/        Library, Reader, Sync
  lib/          db (Dexie), sync (cloud), settings, uiStore, parseFile, useAutoHideHeader
  workers/      parser.worker.ts (encoding + chapter detection)
worker/         Cloudflare Worker API and D1 schema
public/         manifest, service worker, icons
```

## API

All routes live under `/api/books` and require `Authorization: Bearer <API_TOKEN>`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/books` | List synced books |
| POST | `/api/books` | Create or update book metadata |
| GET / DELETE | `/api/books/:id` | Get or delete one book |
| GET / PUT | `/api/books/:id/chapters` | Chapter index (JSON) |
| GET / PUT | `/api/books/:id/text` | Book text (gzip, stored in KV) |
| PUT | `/api/books/:id/progress` | Reading position |

A book only becomes visible to other devices after its text upload finishes.

## Security notes

- The token is stored **only** as a Cloudflare secret and in each device's `localStorage`. It is not in the repo.
- If a token is ever exposed, rotate it by running `npx wrangler secret put API_TOKEN` again and re-entering it on each device.
- The IDs in `wrangler.jsonc` are identifiers, not credentials. They are useless without the token.
- This is a single-user design: one shared token, no user accounts.

## Limits (Cloudflare free tier)

- KV allows about 1,000 writes per day, which is plenty for one reader.
- A KV value can be at most 25 MiB. Text is gzipped before upload, and the Worker rejects anything over 24 MiB.
- D1 rows are capped at 2 MB, so a book's chapter index must stay under about 1.8 MB of JSON.

## Ideas for later

- Bookmarks and highlights
- Keep the screen awake while reading (Wake Lock API)
- Text-to-speech
- Virtualized chapter list for books with thousands of chapters
- Reading statistics

## License

Personal project, shared for reference. No license is granted for reuse or redistribution unless one is added to this repository.
