# 🧩 SKILL — AliExpress Chrome Extension — PART 2

> Continued from [PART1](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART1.md) — Updated 2026-05-07 (aligned with SC27 code)

---

## 5. BACKGROUND SERVICE WORKER

### 5.1 File `src/background/bg.ts`

The service worker is the brain of the extension. It manages:
- Message dispatching (IMPORT_PRODUCT, CHECK_AUTH, GET_CONFIG, PRODUCT_DETECTED)
- API calls to the backend
- The extension icon badge
- Chrome notifications

#### Message dispatching

```typescript
chrome.runtime.onMessage.addListener(
  (message: { type: string; payload?: unknown }, _sender, sendResponse) => {
    if (message.type === 'IMPORT_PRODUCT') {
      importProduct(message.payload as ScrapedProduct)
        .then(sendResponse)
        .catch((err) => sendResponse({
          success: false,
          error: err instanceof Error ? err.message : String(err)
        }));
      return true; // ⚠️ MANDATORY for asynchronous response
    }
    if (message.type === 'CHECK_AUTH') {
      checkAuth().then(sendResponse);
      return true;
    }
    if (message.type === 'GET_CONFIG') {
      getConfig().then(sendResponse);
      return true;
    }
    if (message.type === 'PRODUCT_DETECTED') {
      chrome.action.setBadgeText({ text: '1' });
      chrome.action.setBadgeBackgroundColor({ color: '#7c3aed' });
      return false; // Synchronous
    }
    return false;
  }
);
```

> **⚠️ Critical rule**: `return true` is MANDATORY when `sendResponse` is called asynchronously (inside a `.then()`). Without this, the message port closes before the response is sent.

#### Reset badge on navigation

```typescript
chrome.tabs.onUpdated.addListener((tabId, changeInfo) => {
  if (changeInfo.status === 'loading') {
    chrome.action.setBadgeText({ text: '', tabId });
  }
});
```

### 5.2 `importProduct()` function

```typescript
async function importProduct(product: ScrapedProduct): Promise<ImportResult> {
  const config = await getConfig();

  // Config validation
  if (!config.token) {
    return { success: false, error: 'SC27 token missing. Configure SC GO in options.' };
  }
  if (!config.backendUrl) {
    return {
      success: false,
      error: 'SC27 backend URL missing. Configure SC GO in options.',
    };
  }

  // Calculate selling price with margin
  const sellingPrice = Math.ceil(product.priceMin * config.defaultMargin);

  const body = {
    productId: product.productId,
    title: product.title,
    price: { min: product.priceMin, max: product.priceMax, currency: product.currency },
    mainImage: product.mainImage,
    images: product.images,
    description: product.description,
    skus: product.skus,
    storeId: product.storeId,
    storeName: product.storeName,
    rating: product.rating,
    reviewCount: product.reviewCount,
    url: product.url,
    sellingPrice,
    source: 'extension',
  };

  const res = await fetch(`${config.backendUrl}/api/admin/aliexpress/import-extension`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${config.token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });

  const data = await res.json();
  if (!res.ok || data['error']) {
    return { success: false, error: String(data['error'] || `HTTP ${res.status}`) };
  }

  // Native Chrome notification
  await chrome.notifications.create({
    type: 'basic',
    iconUrl: 'icons/icon48.png',
    title: 'Product imported!',
    message: `"${product.title.slice(0, 60)}" added to the catalog.`,
  });

  return {
    success: true,
    pendingId: String(data['pendingId'] || ''),
    adminUrl: `${config.backendUrl}/admin?tab=aliexpress#pending`,
  };
}
```

### 5.3 `checkAuth()` function

```typescript
async function checkAuth(): Promise<AuthStatus> {
  const config = await getConfig();
  if (!config.token || !config.backendUrl) {
    return { authenticated: false, error: 'Not configured' };
  }
  try {
    const res = await fetch(`${config.backendUrl}/api/admin/aliexpress/status`, {
      headers: { Authorization: `Bearer ${config.token}` },
    });
    if (res.ok) return { authenticated: true, backendUrl: config.backendUrl };
    return { authenticated: false, error: `HTTP ${res.status}` };
  } catch (err) {
    return { authenticated: false, error: err instanceof Error ? err.message : String(err) };
  }
}
```

### 5.4 `getConfig()` function

```typescript
const DEFAULT_BACKEND = 'https://YOUR_BACKEND_DOMAIN';

async function getConfig(): Promise<ScGoConfig> {
  const result = await chrome.storage.local.get([
    'scgo_token', 'scgo_backend_url', 'scgo_margin'
  ]);
  return {
    token: String(result['scgo_token'] || ''),
    backendUrl: String(result['scgo_backend_url'] || DEFAULT_BACKEND),
    defaultMargin: parseFloat(String(result['scgo_margin'] || '2.5')),
  };
}
```

### 5.5 `chrome.storage.local` keys

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `scgo_token` | string | `''` | Bearer admin token |
| `scgo_backend_url` | string | `https://YOUR_BACKEND_DOMAIN` | Complete backend URL (without trailing slash) |
| `scgo_margin` | number | `2.5` | Price multiplier (e.g., ×2.5) |

---

## 6. POPUP UI (3 STATES)

### 6.1 Popup states

| State | Condition | Display |
|------|-----------|-----------|
| `unconfigured` | `!auth.authenticated` | Logo + message + "Configure" button |
| `product` | On AliExpress page + product detected | Product preview + "Import" button |
| `idle` | Authenticated but not on product page | Today's stats + catalog link + options button |

### 6.2 HTML structure `popup.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <link rel="stylesheet" href="popup.css" />
</head>
<body>
  <div id="app">
    <!-- State 1: Unconfigured -->
    <div id="state-unconfigured" class="state" hidden>
      <div class="logo-block">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
      </div>
      <p class="desc">Configure the extension to import products.</p>
      <button id="btn-open-options" class="btn btn-primary">⚙️ Configure</button>
    </div>

    <!-- State 2: Product detected -->
    <div id="state-product" class="state" hidden>
      <div class="header">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
        <span id="status-dot" class="status-dot status-dot--connected"></span>
      </div>
      <div id="product-preview" class="product-preview">
        <img id="product-img" src="" alt="" class="product-thumb" />
        <div class="product-info">
          <div id="product-title" class="product-title"></div>
          <div id="product-price" class="product-price"></div>
          <div id="product-store" class="product-store"></div>
        </div>
      </div>
      <button id="btn-import" class="btn btn-primary btn-import">
        <span class="btn-icon">🚀</span>
        <span class="btn-label">Import</span>
      </button>
      <div id="import-feedback" class="import-feedback" hidden></div>
      <a id="link-admin" href="#" target="_blank" class="link-admin" hidden>
        → View in admin
      </a>
    </div>

    <!-- State 3: Idle (not on AliExpress) -->
    <div id="state-idle" class="state" hidden>
      <div class="header">
        <span class="logo-icon">✦</span>
        <span class="logo-text">SC GO</span>
        <span class="status-dot status-dot--connected"></span>
      </div>
      <p class="hint">Navigate to an AliExpress product page to import.</p>
      <div class="stats">
        <div class="stat-item">
          <span id="stat-today" class="stat-value">—</span>
          <span class="stat-label">imports today</span>
        </div>
      </div>
      <div class="actions">
        <a id="link-catalog" href="#" target="_blank" class="btn btn-secondary">📦 Catalog</a>
        <button id="btn-options" class="btn btn-ghost">⚙️</button>
      </div>
    </div>
  </div>
  <script src="popup.js" type="module"></script>
</body>
</html>
```

### 6.3 `popup.ts` logic

```typescript
async function init() {
  const auth = await chrome.runtime.sendMessage({ type: 'CHECK_AUTH' }) as AuthStatus;
  const config = await chrome.runtime.sendMessage({ type: 'GET_CONFIG' }) as { backendUrl: string };

  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  const isAliExpress = /aliexpress\.com\/item\/\d+/.test(tab?.url || '');

  if (!auth.authenticated) {
    showState('unconfigured');
    document.getElementById('btn-open-options')?.addEventListener('click', () => {
      chrome.runtime.openOptionsPage();
    });
    return;
  }

  if (isAliExpress && tab?.id) {
    try {
      detectedProduct = await chrome.tabs.sendMessage(tab.id, { type: 'GET_PRODUCT' });
    } catch { detectedProduct = null; }

    if (detectedProduct?.productId) {
      showState('product');
      renderProductPreview(detectedProduct);
      setupImportButton(detectedProduct);
    } else {
      showState('idle');
    }
  } else {
    showState('idle');
  }
}

function showState(name: 'unconfigured' | 'product' | 'idle') {
  document.querySelectorAll<HTMLElement>('.state').forEach(el => el.hidden = true);
  document.getElementById(`state-${name}`)!.hidden = false;
}
```

---

## 7. OPTIONS PAGE

### 7.1 Configuration fields

| Field | Type | Description |
|-------|------|-------------|
| Backend URL | `<input type="url">` | Complete URL of the deployed backend |
| Admin/API Token | `<input type="password">` | Bearer admin token |
| Default margin | `<input type="range" min="1.2" max="5" step="0.1">` | Price multiplier (×1.2 to ×5.0) |

### 7.2 Features

- **Paste Button** 📋: `navigator.clipboard.readText()` to paste the token
- **Visibility Toggle** 👁: toggle the token field between `password` and `text`
- **Save** 💾: write to `chrome.storage.local`
- **Connection Test** 🔗: `GET /api/admin/aliexpress/status` with the Bearer token

### 7.3 `options.ts` logic

```typescript
async function save() {
  const backendUrl = (document.getElementById('backend-url') as HTMLInputElement)
    .value.trim().replace(/\/$/, ''); // Remove trailing slash
  const token = (document.getElementById('token') as HTMLInputElement).value.trim();
  const margin = parseFloat((document.getElementById('margin') as HTMLInputElement).value);

  await chrome.storage.local.set({
    scgo_backend_url: backendUrl,
    scgo_token: token,
    scgo_margin: margin,
  });
  showFeedback('✅ Configuration saved!', 'success');
}

async function testConnection() {
  const data = await chrome.storage.local.get(['scgo_token', 'scgo_backend_url']);
  if (!data['scgo_token']) {
    showFeedback('❌ Token missing.', 'error');
    return;
  }
  const res = await fetch(`${data['scgo_backend_url']}/api/admin/aliexpress/status`, {
    headers: { Authorization: `Bearer ${data['scgo_token']}` },
  });
  if (res.ok) showFeedback('✅ Connection successful!', 'success');
  else showFeedback(`❌ HTTP Error ${res.status}`, 'error');
}
```

---

## 8. SHARED TYPES

### 8.1 File `src/shared/types.ts`

```typescript
export interface ScrapedProduct {
  productId: string;
  title: string;
  priceMin: number;
  priceMax: number;
  currency: string;
  mainImage: string;
  images: string[];
  description: string;
  skus: ScrapedSku[];
  storeId: string;
  storeName: string;
  storeUrl: string;
  rating: number;
  reviewCount: number;
  url: string;
  scrapedAt: string; // ISO 8601
}

export interface ScrapedSku {
  skuId: string;
  sku?: string;
  skuCode?: string;
  sku_code?: string;
  name: string;        // E.g., "Red · XL"
  price: number;
  stock: number;
  quantity?: number;
  available?: boolean;
  image?: string;
  attributes: Record<string, string>; // E.g., { "Color": "Red", "Size": "XL" }
}

export interface ScGoConfig {
  backendUrl: string;
  token: string;
  defaultMargin: number; // Multiplier e.g., 2.5
}

export type MessageType =
  | { type: 'IMPORT_PRODUCT'; payload: ScrapedProduct }
  | { type: 'CHECK_AUTH' }
  | { type: 'GET_CONFIG' }
  | { type: 'GET_PRODUCT' }
  | { type: 'PRODUCT_DETECTED'; payload: ScrapedProduct | null };

export interface ImportResult {
  success: boolean;
  pendingId?: string;
  error?: string;
  adminUrl?: string;
}

export interface AuthStatus {
  authenticated: boolean;
  backendUrl?: string;
  error?: string;
}
```

---

## 9. BUILD SYSTEM (VITE)

### 9.1 `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: 'src',
  build: {
    outDir: '../dist',
    emptyOutDir: true,
    sourcemap: false,
    rollupOptions: {
      input: {
        background: resolve(__dirname, 'src/background/bg.ts'),
        content:    resolve(__dirname, 'src/content/content.ts'),
        popup:      resolve(__dirname, 'src/popup/popup.html'),
        options:    resolve(__dirname, 'src/options/options.html'),
      },
      output: {
        entryFileNames: (chunk) => {
          const map: Record<string, string> = {
            background: 'background.js',
            content: 'content.js',
            popup: 'popup.js',
            options: 'options.js',
          };
          return map[chunk.name] || '[name].js';
        },
        assetFileNames: (asset) => {
          if (asset.name?.endsWith('.css')) {
            const map: Record<string, string> = {
              'content.css': 'content.css',
              'popup.css': 'popup.css',
              'options.css': 'options.css',
            };
            return map[asset.name] || '[name][extname]';
          }
          return 'assets/[name][extname]';
        },
      },
    },
  },
});
```

### 9.2 Critical Vite points for Chrome extensions

| Rule | Explanation |
|-------|-------------|
| `root: 'src'` | Source code is in `src/` |
| `outDir: '../dist'` | Build output goes to `dist/` at root |
| `sourcemap: false` | No sourcemaps in production |
| `entryFileNames` custom | JS files must be at the root of `dist/` (not in sub-folders) for `manifest.json` to find them |
| Multi-entry | 4 entries: background, content, popup, options |

### 9.3 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "types": ["chrome"]
  },
  "include": ["src", "vite.config.ts"]
}
```

### 9.4 `package.json`

```json
{
  "name": "sc-go",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite build --watch",
    "build": "vite build && node --input-type=module -e \"import{copyFileSync}from'fs';copyFileSync('src/content/content.css','dist/content.css');console.log('content.css copied')\"",
    "pack": "node scripts/pack.mjs"
  },
  "devDependencies": {
    "@types/chrome": "^0.0.321",
    "typescript": "~5.8.2",
    "vite": "^7.3.2"
  }
}
```

### 9.5 Commands

```bash
# Development (watch mode — automatic rebuild)
npm run dev

# Production build
npm run build

# ZIP packaging for distribution
npm run pack
```

---

## 10. PACKAGING & DISTRIBUTION

### 10.1 Script `scripts/pack.mjs`

```javascript
import { execSync } from 'child_process';
import { cpSync, mkdirSync } from 'fs';
import { join, resolve } from 'path';

const ROOT = resolve(import.meta.dirname, '..');
const DIST = join(ROOT, 'dist');
const PKG  = join(ROOT, 'package');

// Clean
mkdirSync(PKG, { recursive: true });

// Copy dist/ → package/
cpSync(DIST, PKG, { recursive: true });

// Copy manifest.json
cpSync(join(ROOT, 'manifest.json'), join(PKG, 'manifest.json'));

// Copy icons
cpSync(join(ROOT, 'public', 'icons'), join(PKG, 'icons'), { recursive: true });

// Create ZIP
const version = JSON.parse(readFileSync(join(ROOT, 'package.json'), 'utf8')).version;
execSync(`powershell Compress-Archive -Path "${PKG}\\*" -DestinationPath "${join(ROOT, `sc-go-v${version}.zip`)}" -Force`);
```

### 10.2 Developer Mode Installation

1. `npm run build`
2. Open `chrome://extensions`
3. Enable **Developer mode**
4. Click **Load unpacked**
5. Select the `dist/` folder

---

## 11. BACKEND API (RECEIVING ENDPOINT)

### 11.1 Required Endpoints

| Method | Route | Auth | Description |
|---------|-------|------|-------------|
| `POST` | `/api/admin/aliexpress/import-extension` | Bearer admin token | Receives a scraped product and adds it as pending |
| `GET` | `/api/admin/aliexpress/status` | Bearer admin token | Verifies that the token is valid |

### 11.2 POST `import-extension` Body

```json
{
  "productId": "1234567890",
  "title": "Wireless Bluetooth Headphones",
  "price": { "min": 12.50, "max": 18.90, "currency": "USD" },
  "mainImage": "https://cdn.example.com/product-main.jpg",
  "images": ["https://cdn.example.com/1.jpg", "https://cdn.example.com/2.jpg"],
  "description": "...",
  "skus": [
    {
      "skuId": "12345",
      "sku": "SUPPLIER-SKU-001",
      "skuCode": "SUPPLIER-SKU-001",
      "sku_code": "SUPPLIER-SKU-001",
      "name": "Black · M",
      "price": 14.20,
      "stock": 500,
      "quantity": 500,
      "available": true,
      "image": "https://cdn.example.com/sku.jpg",
      "attributes": { "Color": "Black", "Size": "M" }
    }
  ],
  "storeId": "9876",
  "storeName": "TechStore Official",
  "rating": 4.8,
  "reviewCount": 2340,
  "url": "https://www.aliexpress.com/item/1234567890.html",
  "sellingPrice": 32,
  "source": "extension"
}
```

### 11.3 Expected Response

```json
// Success
{ "success": true, "pendingId": "uuid-xxx" }

// Error
{ "error": "Invalid admin token" }
```

### 11.4 Actual Backend Contract (SC27)

- Auth:
  - Bearer admin token mandatory
  - 401 Unauthorized if token is invalid
- Validation:
  - `productId`, `title`, `mainImage`, `url` mandatory
  - `skus[]` accepts `skuId`, `sku/skuCode/sku_code`, `stock`, `quantity`, `available`
- Persistence:
  - Upsert on `aliexpress_pending_products` via `ON CONFLICT (aliexpress_product_id)`
  - Raw scrape data stored in `product_detail` (JSONB)
  - JSON response: `{ success: true, pendingId }`

### 11.5 SQL Table to Use

The target table is `aliexpress_pending_products` (not `aliexpress_pending`).

Key expected fields:
- `aliexpress_product_id`
- `original_title`
- `original_price_eur`
- `original_images`
- `custom_title`
- `selling_price_eur`
- `images`
- `product_detail`
- `store_name`
- `store_id`
- `status`
- `source`

---

## 12. DEPLOYMENT CHECKLIST

### 12.1 Before Publication

- [ ] Replace `host_permissions` in `manifest.json` with production domains
- [ ] Generate PNG icons (16, 32, 48, 128px) — PNG format mandatory
- [ ] Configure backend endpoint `POST /api/admin/aliexpress/import-extension`
- [ ] Configure backend endpoint `GET /api/admin/aliexpress/status`
- [ ] Verify the `aliexpress_pending_products` table in the database
- [ ] Test the scraper on 10+ different AliExpress products
- [ ] Test the 3 fallback levels of the scraper
- [ ] Verify that `npm run build` produces a functional `dist/`
- [ ] Load in developer mode and test full import
- [ ] Verify Chrome notifications

### 12.2 Scraper Maintenance

> ⚠️ **AliExpress regularly modifies its DOM and JS structures.**

Elements to monitor:
1. Global variable names (`runParams`, `__AE_DATA__`, `_dida_config_`)
2. Module names (`titleModule`, `priceModule`, `skuModule`, etc.)
3. CSS selectors for DOM fallback (Level 3)
4. URL formats (`/item/XXXXX.html`)

### 12.3 Security

| Aspect | Measure |
|--------|--------|
| Admin token | Stored in `chrome.storage.local` (never exposed to the DOM) |
| HTTPS | All requests via HTTPS only |
| CORS | The backend must authorize the Chrome extension's origin |
| Validation | The backend must ALWAYS revalidate received data |
| Rate limiting | Implement on backend (max 60 imports/hour) |

### 12.4 Commands Summary

```bash
# Install dependencies
npm install

# Dev with hot-reload
npm run dev

# Production build
npm run build

# ZIP packaging
npm run pack
```

---

## 📐 DESIGN SYSTEM

### Color Palette

| Usage | Color | Code |
|-------|---------|------|
| Primary gradient | Violet | `#7c3aed → #4f46e5` |
| Success | Green | `#059669 → #10b981` |
| Error | Red | `#dc2626 → #ef4444` |
| Background (popup/options) | Deep black | `#0f0f1a` / `#0a0a14` |
| Main text | Light gray | `#e2e8f0` |
| Secondary text | Gray | `#64748b` |
| Accent | Light violet | `#a78bfa` |

### Typography

- **Font**: `-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
- **Weights**: 600 (semi-bold), 700 (bold), 800 (extra-bold)
- **Popup**: fixed width `320px`, min-height `200px`

---

> **End of skill** — Back to [PART1](./SKILL_CHROME_EXTENSION_ALIEXPRESS_PART1.md)
