# 🧩 SKILL — Chrome Extension: AliExpress Product Importer

> **Version**: 1.1.0 | **Last Updated**: 2026-05-07
> **Project sub-folder**: `sc-go/`

---

## 📋 TABLE OF CONTENTS

| # | Section | File |
|---|---------|---------|
| 1 | Objective & Architecture | PART1 (this file) |
| 2 | Manifest V3 & Configuration | PART1 |
| 3 | AliExpress Scraper (3 levels) | PART1 |
| 4 | Content Script & UI Injection | PART1 |
| 5 | Background Service Worker | PART2 |
| 6 | Popup UI | PART2 |
| 7 | Options Page | PART2 |
| 8 | Shared Types | PART2 |
| 9 | Build System (Vite) | PART2 |
| 10 | Packaging & Distribution | PART2 |
| 11 | Backend API (Receiving endpoint) | PART2 |
| 12 | Deployment Checklist | PART2 |

---

## 1. OBJECTIVE & ARCHITECTURE

### 1.1 Goal

Create a Chrome extension (Manifest V3) that:
1. Automatically detects AliExpress product pages
2. Extracts all product data (title, price, images, variants/SKUs, seller, reviews)
3. Injects an "Add to catalog" floating button directly on the AliExpress page
4. Sends data to the shop backend via a secure REST API (Bearer token)
5. Displays a popup with a preview of the detected product + import button
6. Offers an options page to configure the backend URL, the admin API token, and the default margin

### 1.2 Architecture

```
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
│   │   └── types.ts           ← Shared interfaces
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
│       ├── options.html       ← configuration page
│       ├── options.ts         ← options logic
│       └── options.css        ← options styles
└── dist/                      ← build output (loaded in Chrome)
```

### 1.3 Data Flow

```
┌────────────────────────────────────────────────────────┐
│  ALIEXPRESS PAGE (content script)                      │
│                                                        │
│  1. waitForProductReady() → DOM ready                   │
│  2. scrapeAliExpressProduct() → 3 fallback levels      │
│     ├── Level 1: window.runParams / __AE_DATA__        │
│     ├── Level 2: <script type="application/json">     │
│     └── Level 3: Pure DOM extraction (selectors)       │
│  3. injectButton() → violet floating button            │
│  4. sendMessage('PRODUCT_DETECTED') → badge "1"        │
└──────────────┬─────────────────────────────────────────┘
               │ chrome.runtime.sendMessage
               ▼
┌────────────────────────────────────────────────────────┐
│  BACKGROUND SERVICE WORKER (bg.ts)                     │
│                                                        │
│  • onMessage listener (dispatch by type)              │
│  • IMPORT_PRODUCT → importProduct()                    │
│    ├── getConfig() from chrome.storage.local           │
│    ├── Selling price calculation = priceMin × margin   │
│    ├── POST /api/admin/aliexpress/import-extension     │
│    │   Headers: Authorization: Bearer <token>          │
│    └── chrome.notifications.create (success)           │
│  • CHECK_AUTH → GET /api/admin/aliexpress/status       │
│  • GET_CONFIG → chrome.storage.local.get               │
└──────────────┬─────────────────────────────────────────┘
               │ fetch()
               ▼
┌────────────────────────────────────────────────────────┐
│  SHOP BACKEND (API Vercel/Express)                     │
│                                                        │
│  POST /api/admin/aliexpress/import-extension           │
│  GET  /api/admin/aliexpress/status                     │
│                                                        │
│  → Admin token validation → Insert DB → Return pendingId │
└────────────────────────────────────────────────────────┘
```

---

## 2. MANIFEST V3 — COMPLETE CONFIGURATION

### 2.1 File `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "SC GO — Product Importer",
  "short_name": "SC GO",
  "version": "1.0.0",
  "description": "Import AliExpress products directly into your shop in 1 click.",
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  },
  "background": {
    "service_worker": "background.js",
    "type": "module"
  },
  "content_scripts": [
    {
      "matches": [
        "https://www.aliexpress.com/item/*",
        "https://aliexpress.com/item/*",
        "https://fr.aliexpress.com/item/*",
        "https://www.aliexpress.us/item/*"
      ],
      "js": ["content.js"],
      "css": ["content.css"],
      "run_at": "document_idle"
    }
  ],
  "action": {
    "default_popup": "popup/popup.html",
    "default_title": "SC GO — Product Importer",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "options_page": "options/options.html",
  "permissions": ["storage", "activeTab", "notifications"],
  "host_permissions": [
    "https://YOUR_BACKEND_DOMAIN/*",
    "https://localhost:*/*"
  ]
}
```

### 2.2 Critical points of the manifest

| Field | Explanation |
|-------|-------------|
| `manifest_version: 3` | Mandatory for new Chrome extensions (MV2 deprecated) |
| `background.type: "module"` | Allows ES Module imports in the service worker |
| `content_scripts.matches` | Pattern match on the 4 AliExpress product URL variants |
| `content_scripts.run_at: "document_idle"` | Injection after the DOM is fully loaded |
| `permissions.storage` | Access to `chrome.storage.local` for configuration |
| `permissions.activeTab` | Access to the active tab to communicate with the content script |
| `permissions.notifications` | System notifications after a successful import |
| `host_permissions` | **ADAPT**: set the actual URL of the deployed backend |

### 2.3 Rules for `host_permissions`

- Always add the backend domain (production + staging)
- Format: `https://mywebsite.com/*` (with wildcard path)
- NEVER use `<all_urls>` (rejected by Chrome Web Store)
- localhost is only useful in development

---

## 3. ALIEXPRESS SCRAPER — 3 FALLBACK LEVELS

### 3.1 Strategy

AliExpress frequently changes its DOM and JS structures. The scraper uses **3 fallback levels** to maximize reliability:

| Level | Source | Reliability | Data |
|-------|--------|-----------|---------|
| 1 | `window.runParams` / `__AE_DATA__` / `_dida_config_` | ⭐⭐⭐ High | Complete (SKUs, price, store, reviews) |
| 2 | Embedded JSON `<script>` tags | ⭐⭐ Medium | Complete if JSON is found |
| 3 | Pure DOM extraction (CSS selectors) | ⭐ Low | Partial (no SKUs) |

### 3.2 File `src/content/scraper.ts`

```typescript
// Entry point — 3-level cascade
export function scrapeAliExpressProduct(): ScrapedProduct | null {
  return scrapeFromWindowData() || scrapeFromScriptTags() || scrapeFromDom();
}
```

### 3.3 Level 1: Global JS variables

AliExpress exposes its data in global JavaScript variables:
- `window.runParams.data` (old format)
- `window.__AE_DATA__` (new React/Next format)
- `window._dida_config_` (alternative format)

```typescript
function scrapeFromWindowData(): ScrapedProduct | null {
  const w = window as Record<string, unknown>;
  const runParams = w['runParams'] as Record<string, unknown> | undefined;
  const data = runParams?.['data'] as Record<string, unknown> | undefined;
  const aeData = w['__AE_DATA__'] as Record<string, unknown> | undefined;
  const didaConfig = w['_dida_config_'] as Record<string, unknown> | undefined;
  const source = data || aeData || didaConfig;
  if (!source) return null;
  return extractFromDataObject(source);
}
```

### 3.4 Extraction from structured data object

AliExpress modules follow a predictable structure:

```typescript
function extractFromDataObject(data: Record<string, unknown>): ScrapedProduct | null {
  // Structured AliExpress modules — recursive search
  const titleModule     = findDeep(data, 'titleModule');
  const priceModule     = findDeep(data, 'priceModule');
  const imageModule     = findDeep(data, 'imageModule');
  const skuModule       = findDeep(data, 'skuModule');
  const storeModule     = findDeep(data, 'storeModule');
  const feedbackModule  = findDeep(data, 'feedbackModule');
  const descModule      = findDeep(data, 'descriptionModule');Outfit

  // Title extraction
  const title = titleModule?.['subject'] || titleModule?.['title'] || '';

  // Price extraction (multiple formats possible)
  const priceMin = priceModule?.['minAmount']?.['value']
    || priceModule?.['formatedPrice']
    || priceModule?.['minActivityAmount']?.['value']
    || '0';

  // Images extraction
  const imageList = imageModule?.['imagePathList']
    || imageModule?.['imagePaths']
    || [];

  // SKUs/variants extraction
  const skus = extractSkusFromModule(skuModule);

  // Return complete ScrapedProduct object
  return { productId, title, priceMin, priceMax, currency, mainImage,
           images, description, skus, storeId, storeName, storeUrl,
           rating, reviewCount, url, scrapedAt };
}
```

### 3.5 Variants (SKUs) extraction

```typescript
function extractSkusFromModule(skuModule): ScrapedSku[] {
  const skuList = skuModule['skuPriceList'] || [];
  const props = skuModule['productSKUPropertyList'] || [];

  return skuList.map((sku) => {
    const skuPropIds = String(sku['skuPropIds'] || '').split(',');
    const attributes: Record<string, string> = {};

    // Resolve attributes (color, size, etc.)
    for (const prop of props) {
      const propValues = prop['skuPropertyValues'] || [];
      const propName = String(prop['skuPropertyName'] || '');
      for (const val of propValues) {
        if (skuPropIds.includes(String(val['propertyValueId']))) {
          attributes[propName] = String(
            val['propertyValueDisplayName'] || val['propertyValueName']
          );
        }
      }
    }

    // Variant price
    const skuPrice = sku['skuVal'];
    const price = parseFloat(
      skuPrice?.['actSkuCalPrice'] || skuPrice?.['skuCalPrice'] || '0'
    );
    // Actual supplier quantity (no more 999 fallback)
    const rawQty =
      skuPrice?.['skuQuantity']
      ?? skuPrice?.['availQuantity']
      ?? sku['skuStock']
      ?? sku['availQuantity']
      ?? null;
    const parsedQty = Number.parseInt(String(rawQty ?? ''), 10);
    const quantity = Number.isFinite(parsedQty) && parsedQty >= 0 ? parsedQty : undefined;

    // Actual supplier availability
    const rawAvailability =
      skuPrice?.['isAvailableForSale']
      ?? skuPrice?.['available']
      ?? sku['available']
      ?? sku['isAvailable']
      ?? null;
    const available =
      typeof rawAvailability === 'boolean'
        ? rawAvailability
        : typeof rawAvailability === 'string'
          ? ['1', 'true', 'yes', 'on'].includes(rawAvailability.trim().toLowerCase())
          : quantity != null
            ? quantity > 0
            : undefined;

    // Supplier source SKU (seller SKU)
    const supplierSkuCandidates = [
      sku['sellerSku'], sku['seller_sku'], sku['skuCode'], sku['sku_code'],
      skuPrice?.['sellerSku'], skuPrice?.['seller_sku'], skuPrice?.['skuCode'], skuPrice?.['sku_code'],
    ];
    const supplierSku = supplierSkuCandidates
      .map((candidate) => String(candidate || '').trim())
      .find((candidate) => candidate.length > 0);

    const stock = quantity ?? 0;

    return {
      skuId: String(sku['skuId']),
      sku: supplierSku || undefined,
      skuCode: supplierSku || undefined,
      sku_code: supplierSku || undefined,
      name: Object.values(attributes).join(' · ') || 'Variant',
      price, stock,
      quantity,
      available,
      image: sku['skuImage'] || undefined,
      attributes,
    };
  });
}
```

### 3.6 Level 2: JSON `<script>` tags

```typescript
function scrapeFromScriptTags(): ScrapedProduct | null {
  const scripts = Array.from(
    document.querySelectorAll('script[type="application/json"], script:not([src])')
  );
  for (const script of scripts) {
    const text = script.textContent || '';
    if (text.includes('skuModule') || text.includes('titleModule')
        || text.includes('priceModule')) {
      try {
        const parsed = JSON.parse(text);
        const result = extractFromDataObject(parsed);
        if (result) return result;
      } catch { /* Invalid JSON, continue */ }
    }
  }
  return null;
}
```

### 3.7 Level 3: Pure DOM extraction

CSS selectors used (subject to change — must be maintained):

| Data | CSS selectors |
|------|---------------|
| **Title** | `h1[data-pl="product-title"]`, `.product-title-text`, `[class*="title--"] h1`, `h1.product-title` |
| **Price** | `[class*="price--"] [class*="current"]`, `[class*="uniform-banner-box-price"]`, `.product-price-value`, `[data-pl="product-price"]` |
| **Main image** | `[class*="magnifier"] img`, `.product-image img`, `meta[property="og:image"]` |
| **Gallery images** | `[class*="slider--"] img`, `.images-view-list img`, `[class*="gallery"] img` |
| **Shop** | `[class*="shop-name"]`, `[class*="store-header"] a` |
| **Rating** | `[class*="score--"]`, `[class*="overview-rating"]` |
| **Review count** | `[class*="quantity--"]`, `[class*="review-count"]` |

### 3.8 Scraper helpers

```typescript
// Extract product ID from URL: /item/1234567890.html → "1234567890"
function extractProductIdFromUrl(): string {
  const match = window.location.pathname.match(/\/item\/(\d+)\.html/);
  return match?.[1] || '';
}

// Recursive search for a key in a nested object (max 8 levels)
function findDeep(obj: unknown, key: string, depth = 0): unknown {
  if (depth > 8) return null;
  if (typeof obj !== 'object' || obj === null) return null;
  const record = obj as Record<string, unknown>;
  if (key in record) return record[key];
  for (const v of Object.values(record)) {
    const found = findDeep(v, key, depth + 1);
    if (found !== null) return found;
  }
  return null;
}

// High resolution image: remove resizing suffixes
imageUrl.replace(/_\d+x\d+.*?\.jpg/, '.jpg');
```

---

## 4. CONTENT SCRIPT & UI INJECTION

### 4.1 File `src/content/content.ts`

#### Initialization with DOM wait

```typescript
let currentProduct: ScrapedProduct | null = null;
let buttonInjected = false;

// Wait until title + price are in the DOM (max 10s = 20 × 500ms)
function waitForProductReady(callback: () => void, attempts = 0) {
  const hasTitle = !!document.querySelector(
    'h1, [class*="title--"], .product-title-text'
  );
  const hasPrice = !!document.querySelector(
    '[class*="price--"], .product-price-value, [data-pl="product-price"]'
  );
  if (hasTitle && hasPrice) callback();
  else if (attempts < 20) setTimeout(() => waitForProductReady(callback, attempts + 1), 500);
}

function init() {
  waitForProductReady(() => {
    currentProduct = scrapeAliExpressProduct();
    if (currentProduct?.productId) {
      injectButton(currentProduct);
      chrome.runtime.sendMessage({ type: 'PRODUCT_DETECTED', payload: currentProduct });
    }
  });
}
```

#### Floating button injection

```typescript
function injectButton(product: ScrapedProduct) {
  if (buttonInjected) return;
  buttonInjected = true;

  const btn = document.createElement('div');
  btn.id = 'sc-go-btn';
  btn.innerHTML = `
    <button id="sc-go-trigger" title="Add to catalog">
      <span class="sc-go-icon">✦</span>
      <span class="sc-go-label">Add to catalog</span>
    </button>
    <div id="sc-go-toast" class="sc-go-toast"></div>
  `;
  document.body.appendChild(btn);
  document.getElementById('sc-go-trigger')?.addEventListener('click', () => {
    handleImport(product);
  });
}
```

#### Import logic & button states

```typescript
async function handleImport(product: ScrapedProduct) {
  setButtonState('loading');
  const response = await chrome.runtime.sendMessage({
    type: 'IMPORT_PRODUCT', payload: product,
  });
  if (response?.success) {
    setButtonState('success');
    showToast('✅ Added to catalog!', 'success');
    // Direct link to admin
    if (response.adminUrl) {
      const link = document.createElement('a');
      link.href = response.adminUrl;
      link.target = '_blank';
      link.textContent = '→ View in admin';
      document.getElementById('sc-go-btn')?.appendChild(link);
    }
  } else {
    setButtonState('error');
    showToast(`❌ ${response?.error || 'Error'}`, 'error');
    setTimeout(() => setButtonState('idle'), 3000);
  }
}

type ButtonState = 'idle' | 'loading' | 'success' | 'error';

function setButtonState(state: ButtonState) {
  const btn = document.getElementById('sc-go-trigger');
  btn?.setAttribute('data-state', state);
  const labels: Record<ButtonState, string> = {
    idle: 'Add to catalog',
    loading: 'Syncing…',
    success: 'In the catalog!',
    error: 'Error — Try again',
  };
  const label = btn?.querySelector('.sc-go-label');
  if (label) label.textContent = labels[state];
}
```

#### SPA management (navigation without reloading)

```typescript
// Observer to detect AliExpress SPA page changes
const observer = new MutationObserver(() => {
  const isProductPage = /\/item\/\d+\.html/.test(window.location.pathname);
  if (isProductPage && !buttonInjected) init();
});
observer.observe(document.body, { childList: true, subtree: true });
init(); // First launch
```

#### Respond to the popup (GET_PRODUCT)

```typescript
chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message.type === 'GET_PRODUCT') {
    sendResponse(currentProduct); // Synchronous response
    return false;
  }
  return false;
});
```

### 4.2 File `src/content/content.css` — Floating button

```css
/* Fixed container at bottom right, max z-index */
#sc-go-btn {
  position: fixed;
  bottom: 32px;
  right: 32px;
  z-index: 2147483647;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 8px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

/* Main button — violet gradient */
#sc-go-trigger {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 22px;
  background: linear-gradient(135deg, #7c3aed, #4f46e5);
  color: #ffffff;
  border: none;
  border-radius: 50px;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 8px 32px rgba(124, 58, 237, 0.45),
              0 2px 8px rgba(0, 0, 0, 0.15);
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

/* Hover: elevation */
#sc-go-trigger:hover {
  transform: translateY(-2px) scale(1.03);
  box-shadow: 0 12px 40px rgba(124, 58, 237, 0.55);
}

/* Visual states via data-attribute */
#sc-go-trigger[data-state='loading'] {
  background: linear-gradient(135deg, #6d28d9, #4338ca);
  cursor: wait;
  pointer-events: none;
}
#sc-go-trigger[data-state='loading'] .sc-go-icon {
  animation: sc-go-spin 1s linear infinite;
}
#sc-go-trigger[data-state='success'] {
  background: linear-gradient(135deg, #059669, #10b981);
  pointer-events: none;
}
#sc-go-trigger[data-state='error'] {
  background: linear-gradient(135deg, #dc2626, #ef4444);
}

/* Toast notification */
.sc-go-toast {
  display: none; opacity: 0; transform: translateY(4px);
  padding: 12px 18px; border-radius: 12px; font-size: 13px;
  font-weight: 600; max-width: 280px;
  backdrop-filter: blur(8px);
  transition: opacity 0.3s, transform 0.3s;
}
.sc-go-toast--visible { display: block; opacity: 1; transform: translateY(0); }
.sc-go-toast--success { background: rgba(16, 185, 129, 0.95); color: #fff; }
.sc-go-toast--error   { background: rgba(239, 68, 68, 0.95); color: #fff; }
```

---

> **Continued** → [SKILL_CHROME_EXTENSION_ALIEXPRESS_PART2.md](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART2.md)
