# Birthday Herald

Personalised "the day you were born" newspapers sold through Shopify. A customer enters a date of birth, name, message and photo, sees a vintage front page filled with that year's news, songs, prices and famous birthdays, and adds it to the cart. Each order produces a print-ready A3 PDF, rendered in the background.

![Editor with a created newspaper](docs/screenshots/editor-created.jpg)

A full illustrated walkthrough of the project is in [docs/Birthday-Herald-Project-Documentation.pdf](docs/Birthday-Herald-Project-Documentation.pdf).

## Tech stack

| Area | Choice |
| --- | --- |
| Frontend | React 18, React Router 6, Vite 5, Tailwind CSS 3 |
| Photo cut-out | `@imgly/background-removal` (runs in the browser) |
| Backend | Node.js 18+, Express 4 (ES modules) |
| Database | PostgreSQL via Prisma 5 |
| Background jobs | BullMQ on Redis (`pdf-generation` and `data-import` queues) |
| PDF rendering | puppeteer-core + system Chrome, sharp, pdf-lib |
| File storage | Cloudinary (PDFs and admin images) |
| Commerce | Shopify Storefront API (client) and Admin API (server) |
| Hosting | Heroku: `web`, `worker` and `release` processes (see `Procfile`) |

## Getting started

Requires Node.js 18+, PostgreSQL, Redis and Google Chrome.

```bash
# server (API on http://localhost:3001)
cd server
npm install                 # then create server/.env (see "Environment variables")
npx prisma migrate dev
npm run dev

# worker, in a second terminal (renders PDFs, runs imports)
cd server
npm run worker:dev

# client (editor on http://localhost:5173, proxies /api to :3001)
cd client
npm install
npm run dev
```

| Script | Where | What it does |
| --- | --- | --- |
| `npm run dev` | server | API with auto-reload |
| `npm start` | server | API (runs `prisma generate` first) |
| `npm run worker` / `worker:dev` | server | BullMQ worker for PDFs and imports |
| `npm run prisma:migrate` / `prisma:studio` | server | Create/apply migrations, browse the database |
| `npm run dev` / `build` / `preview` | client | Vite dev server, production build into `client/dist`, preview |
| `npm run build` / `npm start` | root | Build the client, start the server (what Heroku runs) |

Without a running worker, PDFs and imports stay queued.

## Project structure

```text
client/src/
├── main.jsx                 Routes: / (editor) and /admin/*
├── App.jsx                  Template choice, year data fetch, scaled preview, PDF render mode
├── Editor.jsx               Customer form, age choice, variants, Create / Add to Cart
├── EditorToolbar.jsx        Photo zoom / rotate / move
├── Newspaper.jsx            Template 1 – "The Birthday Herald"
├── Newspaper1.jsx           Template 2 – "The Birthday Times" (cream)
├── Newspaper2.jsx           Template 3 – "The Birthday Times" (aged paper)
├── Newspaper3.jsx           Template 4 – "The Birthday Times" (grey newsprint)
├── QueueStatus.jsx          Admin queue page
├── admin/                   Admin layout, login, data index, CRUD table, saved newspapers
├── components/              RevealPhoto, PmImage, VintageAdImage, SketchBorder
├── utils/                   age.js, newsFlow.js, imageProcessor.js, text helpers
├── shopify-utils.js         API base URL, save PDF for cart, add to Shopify cart
└── shopify-storefront.js    Storefront API (variants and prices)
client/public/               Paper textures, frames, fonts, sample photos

server/
├── server.js                Web process: API + serves client/dist
├── worker.js                Worker process: PDF generation + data import
├── routes/ controllers/ services/ middleware/
├── jobs/                    queues.js, redis.js, processors/pdfGenerationProcessor.js, dataImportProcessor.js
├── config/                  admin-crud-models.js (editable tables), cloudinary.js, cors.js
├── utils/                   pdf-storage.js, image-processor.js, admin-field-coercion.js
└── prisma/                  schema.prisma, migrations/
```

## Templates

`?temp=` in the URL chooses the template: `1` (or none), `2`, `3` or `4`. Each template shows a built-in sample until the customer enters a date of birth. After that, the preview is filled from `GET /api/year?year=&month=&date=`.

Other URL parameters: `productId` (Shopify product whose variants and prices are shown), `day`, `month` and `year` (prefill the date of birth, 1920–2010 only), and `pdfId` (used by the PDF renderer; loads a saved design).

To add a template:

1. Create `client/src/NewspaperN.jsx` with a root element `id="__birthday_newspaper"`. The PDF renderer captures this element.
2. Map a new `?temp=` value in `App.jsx`, and give it a sample design in `createDefaultInfo()` / `SAMPLE_DATE_BY_TEMPLATE`.
3. Add its Shopify variant IDs in `Editor.jsx` (`TEMPLATE_VARIANT_OPTIONS` / `getTemplateKey`).

## How an order becomes a PDF

1. **Add to Cart** → `POST /api/save-pdf-for-cart` creates a `SavedNewspaper` row (`PENDING`) and queues a BullMQ job whose id is the record id.
2. The item is added to the Shopify cart with hidden properties `_pdf_id`, `_admin_saved_newspaper_url`, `_customer_name` and `_birth_year`.
3. The worker tints the photo (sharp) and marks the record `PROCESSING`. It then opens `<page URL>?pdfId=…` in headless Chrome at 1697 CSS px × 4 device scale (≈ 580 DPI). The page loads the design from `/api/pdf-info/:pdfId`.
4. The sheet is captured in strips, stitched, and JPEG-encoded. The quality steps down from 98 until the file is 8.5–9.8 MB, because Cloudinary's raw upload limit is 10 MB. pdf-lib places it on one A3 page.
5. The PDF is uploaded to Cloudinary and the record becomes `COMPLETED` with `pdfUrl`. Failed jobs are retried 3 times with exponential backoff.
6. Shopify's order webhook (`POST /api/webhooks/order-created`) links the record to the order. It also writes the order metafield `birthday.admin_saved_newspaper_url`, so staff can open the PDF from the order.

Check status with `GET /api/pdf-status/:pdfId`, `/api/checkout-pdf/:pdfId` or `/api/order-pdf/:orderId`.

## Admin panel

`/admin` is protected by a single shared password (`ADMIN_PASSWORD`, 7-day session cookie).

| Page | What it does |
| --- | --- |
| `/admin/queue` | PDF queue counts and jobs, with Retry and Delete |
| `/admin/data` | All tables. Each table has add/edit/delete, image upload to Cloudinary and CSV/XLSX bulk import |
| `/admin/saved-newspapers` | Every design sent to the cart: status, view JSON, download PDF, re-generate |

Only tables listed in `server/config/admin-crud-models.js` are editable. Import formats and examples are in [server/IMPORT_UPLOAD.md](server/IMPORT_UPLOAD.md), and the admin API is described in [server/ADMIN_ENV.md](server/ADMIN_ENV.md).

## Content data

These tables fill the newspaper, looked up by birth year (and by day and month where relevant): `BirthdayTwin`, `FamousBirthday`, `TopSongsOfTheYear`, `NewsEvent`, `CarOfTheYear`, `PM`, `PmMeeting`, `PetrolPrice`, `HouseIncome`, `MedianHousePrice`, `Population` (WORLD / AUSTRALIA), `FunFact`, `ArticleValue`, `VintageAdvertisement`. Designs sent to the cart are stored in `SavedNewspaper`.

A few of the rules:

- Famous birthdays match on day and month (any year), up to 5.
- If a year has no car, the latest car is used.
- The PM is the one in office on the date. The word "PM" in `PmMeeting` stories is replaced with that PM's name.

## Environment variables

**Server** (`server/.env`)

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL`, `REDIS_URL` | Postgres and Redis (Redis uses TLS when `NODE_ENV=production`) |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | File storage |
| `ADMIN_PASSWORD`, `SESSION_SECRET` | Admin login and session signing |
| `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SHOPIFY_ACCESS_TOKEN`, `SHOPIFY_SHOP_DOMAIN`, `SHOPIFY_APP_URL`, `SHOPIFY_SCOPES`, `SHOPIFY_PRIVATE_APP` | Shopify Admin API |
| `APP_URL` | Base URL used when the admin re-generates a PDF |
| `CHROME_BIN` / `PUPPETEER_EXECUTABLE_PATH` | Chrome for the worker (common install paths are also tried) |
| `PDF_TARGET_DPI`, `PDF_JPEG_QUALITY`, `PDF_JOB_ATTEMPTS`, `PDF_JOB_BACKOFF_MS`, `PDF_WORKER_CONCURRENCY` | PDF tuning |
| `DATA_IMPORT_MAX_FILE_MB`, `DATA_IMPORT_BATCH_SIZE`, `DATA_IMPORT_WORKER_CONCURRENCY` | Import tuning |
| `NODE_ENV`, `PORT` | `development` also saves PDFs locally in `server/storage/pdfs` |

**Client** (`client/.env`)

| Variable | Purpose |
| --- | --- |
| `VITE_API_BASE_URL` | API host, if not the same origin |
| `VITE_SHOPIFY_STOREFRONT_TOKEN`, `VITE_SHOPIFY_SHOP_DOMAIN`, `VITE_SHOPIFY_STOREFRONT_API_VERSION` | Storefront API for variant prices |
| `VITE_DEV_API_PROXY_TARGET` | Where the Vite dev server proxies `/api` (default `http://localhost:3001`) |

## Deployment

Heroku, as defined in `Procfile` and `app.json`:

- **web**: `npm start`. Express serves the API and the built client.
- **worker**: `cd server && npm run worker`. Scale with `heroku ps:scale worker=1`.
- **release**: `prisma migrate deploy`, with up to 5 retries.

`heroku-postbuild` builds the client and installs the server without downloading Chromium. Chrome comes from the Puppeteer buildpack. Add-ons: Heroku Postgres and Redis.

Shopify setup guides: [SHOPIFY_SETUP.md](SHOPIFY_SETUP.md), [PRIVATE_APP_SETUP.md](PRIVATE_APP_SETUP.md), [STOREFRONT_INTEGRATION.md](STOREFRONT_INTEGRATION.md) and [API_SCOPES_EXPLAINED.md](API_SCOPES_EXPLAINED.md). The theme snippet is in `shopify-theme-integration.liquid`.

## Known issues

- **Webhook not verified.** `POST /api/webhooks/order-created` does not check Shopify's HMAC signature. The app-proxy signature check is also still a TODO.
- **Re-generate loses the template.** Admin "Generate" re-queues a PDF with the bare `APP_URL`, without `?temp=`, so designs from templates 2–4 are rendered in template 1.
- **CORS allows every origin**, with credentials (`server/config/cors.js`, marked "temporarily for debugging").
- **Hard-coded values.** The Heroku admin URL is fixed in `webhook-controller.js` and `shopify-utils.js`. There are Storefront token and shop fallbacks in `shopify-storefront.js`, an ngrok host in `vite.config.js`, and test variant prices in `Editor.jsx`.
- **Render width mismatch.** The worker assumes a 1697 px sheet (`LAYOUT_WIDTH_CSS_PX`), but `App.jsx` renders it at 1712 px.
- **No automated tests.**
