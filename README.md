# AliExpress Chrome Extension — Technical Documentation

## Objective

This documentation describes the current actual state of a Chrome extension (Manifest V3) designed to:

- detect AliExpress product pages,
- extract product and variant data,
- send this data to a secure import service,
- manage imports via an injected button and a popup.

## Exact Current Features

- Automatic detection of AliExpress product pages.
- Scraping with 3 fallback levels:
  - level 1: global JavaScript data,
  - level 2: embedded JSON blocks,
  - level 3: DOM fallback.
- Advanced variant extraction:
  - attributes (e.g., color, size),
  - variant price,
  - actual quantity,
  - actual availability,
  - supplier SKU (`sku`, `skuCode`, `sku_code`) when available.
- Floating button injection on the product page.
- Button states:
  - `idle`,
  - `loading`,
  - `success`,
  - `error`.
- In-page feedback toasts.
- Full extension messaging:
  - `PRODUCT_DETECTED`,
  - `GET_PRODUCT`,
  - `IMPORT_PRODUCT`,
  - `CHECK_AUTH`,
  - `GET_CONFIG`.
- Network import via background service worker with `Authorization: Bearer <token>` header.
- System notification after successful import.
- Extension badge:
  - set to `1` when a product is detected,
  - reset during navigation.
- Popup with 3 states:
  - unconfigured,
  - product detected,
  - not on a product page.
- Advanced options:
  - local configuration storage,
  - token pasting,
  - show/hide token,
  - margin slider,
  - connection test.
- Multi-entry Vite + TypeScript build.
- Automated ZIP packaging.

## Technical Architecture

1. Content Script (`src/content/content.ts`)
   - waits for the critical DOM (`waitForProductReady`),
   - triggers `scrapeAliExpressProduct()`,
   - injects the button,
   - exposes `GET_PRODUCT`,
   - sends `PRODUCT_DETECTED` to the background.

2. Scraper (`src/content/scraper.ts`)
   - cascade: `window data -> script tags -> DOM`,
   - normalizes product + variants,
   - retrieves product ID from the URL path,
   - builds a server-ready payload.

3. Background (`src/background/bg.ts`)
   - orchestrates runtime messages,
   - runs the network import,
   - runs the auth check,
   - provides config to the popup,
   - manages notifications and the badge.

4. Popup (`src/popup/popup.ts`)
   - selects the UI state,
   - displays the product preview,
   - triggers manual import,
   - displays result feedback.

5. Options (`src/options/options.ts`)
   - loads/saves configuration,
   - manages token and margin,
   - triggers a connection test.

## Project Structure

```text
sc-go/
├── manifest.json              ← Manifest V3
├── package.json               ← npm config (dev deps only)
├── tsconfig.json              ← strict TypeScript
├── vite.config.ts             ← multi-entry build
├── scripts/
│   └── pack.mjs               ← ZIP packaging
├── public/
│   └── icons/                 ← Icons 16/32/48/128px
├── src/
│   ├── shared/
│   │   └── types.ts           ← shared interfaces
│   ├── background/
│   │   └── bg.ts              ← Service Worker (messages, import, auth)
│   ├── content/
│   │   ├── scraper.ts         ← AliExpress DOM extraction (3 fallbacks)
│   │   ├── content.ts         ← button injection + import logic
│   │   └── content.css        ← floating button styles
│   ├── popup/
│   │   ├── popup.html         ← popup UI (3 states)
│   │   ├── popup.ts           ← popup logic
│   │   └── popup.css          ← popup styles
│   └── options/
│       ├── options.html       ← options page
│       ├── options.ts         ← options logic
│       └── options.css        ← options styles
└── dist/                      ← build output (loaded in Chrome)
```

## Detailed Workflow

1. The user opens an AliExpress product page.
2. The content script waits for DOM markers (title + price).
3. The scraper attempts level 1 extraction.
4. In case of partial failure, fall back to level 2, then level 3.
5. If the product is valid:
   - the button is injected,
   - the `PRODUCT_DETECTED` message is sent.
6. The user clicks "Import" (or the floating button).
7. The content script sends `IMPORT_PRODUCT` to the background.
8. The background:
   - reads the local config,
   - calculates `sellingPrice` from `priceMin * margin`,
   - sends the payload to the import service,
   - returns success/error.
9. The content script updates the visual state + feedback.
10. The popup reflects the state and allows an alternative import.

## Data Contract (Summary)

- Product:
  - `productId`, `title`, `priceMin`, `priceMax`, `currency`,
  - `mainImage`, `images`, `description`,
  - `storeId`, `storeName`, shop reference,
  - `rating`, `reviewCount`,
  - source page reference, `scrapedAt`.
- Variants:
  - `skuId`,
  - `sku`, `skuCode`, `sku_code`,
  - `name`, `price`,
  - `stock`, `quantity`, `available`,
  - `image`,
  - `attributes`.

## Available Scripts

From the `sc-go` directory:

```bash
npm install
npm run dev
npm run build
npm run pack
```

Notes:

- `dev`: build in watch mode.
- `build`: extension bundle + explicit copy of `content.css`.
- `pack`: generates a versioned ZIP from build artifacts.

## Extension Installation (Developer Mode)

1. Run `npm run build`.
2. Open the Chrome Extensions page.
3. Enable developer mode.
4. Load the unpacked extension.
5. Select the `dist` folder.

## Robustness Highlights

- Multi-source fallback to withstand AliExpress structure changes.
- DOM Observer to react to dynamic navigations.
- Explicit handling of network/auth errors.
- Immediate user feedback (button state + toast + popup).

## Current Constraints and Limitations

- Scraping quality depends on the stability of the source structure.
- Some product sheets might provide partial data.
- The DOM fallback does not always reconstruct complete variants.

## Security and Documentation Compliance

- No plaintext secrets.
- No real tokens.
- No mention of internal brands or projects.
- No external addresses in this document.
