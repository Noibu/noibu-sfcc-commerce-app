# Noibu Commerce App

Loads the Noibu session monitoring script on every storefront page and tracks ecommerce events via the Noibu SDK.

## Architecture

The app is a **UI-only Commerce App** with no SFCC cartridge or backend component. It integrates with Storefront Next through two mechanisms:

### Script injection — `NoibuProvider`

`NoibuProvider` is registered as a global context provider via `target-config.json`. It renders a `<script async src="https://cdn.noibu.com/collect-core.js">` tag, which React 19 hoists to `<head>` and deduplicates by `src`. This means the script loads exactly once regardless of how many times the provider renders.

### Event tracking — `NoibuAdapter`

`NoibuProvider` registers a custom `EngagementAdapter` on mount via `addAdapter`. The adapter hooks into the storefront's analytics fan-out system: every call to `analytics.track*()` in the storefront is forwarded to all registered adapters, including Noibu.

The adapter maps Storefront Next analytics events to Noibu SDK calls (`window.NOIBUJS.track(eventName, payload)`). Because `collect-core.js` loads asynchronously, the adapter queues calls via `window.addEventListener('noibuSDKReady', ...)` when the SDK is not yet available.

```
Storefront analytics call
        │
        ▼
  EngagementAdapter fan-out (src/lib/adapters)
        │
        ▼
  NoibuAdapter.sendEvent()
        │
        ├─ window.NOIBUJS available ──▶ NOIBUJS.track(eventName, payload)
        │
        └─ not yet available ─────────▶ addEventListener('noibuSDKReady', ...)
```

### File structure

```
sfcc-storefront/src/extensions/noibu/
├── adapters/
│   └── noibu-adapter.ts      # Event mapping and SDK integration
├── providers/
│   └── noibu-provider.tsx    # Script injection + adapter registration
├── tests/
│   ├── noibu-adapter.test.ts
│   └── noibu-provider.test.tsx
├── index.ts                  # Public exports
└── target-config.json        # Extension system registration
```

## Supported Events

| Storefront event | Noibu event | Payload highlights |
|---|---|---|
| `view_product` | `product_viewed` | variant id, sku, title, price, master product id |
| `cart_item_add` | `product_added_to_cart` | merchandise variant, quantity, line total |
| `view_search` | `search_submitted` | search query, result variants |
| `view_category` | `collection_viewed` | category id, title, result variants |
| `checkout_start` | `checkout_started` | currency, subtotal, total, line items |
| `checkout_step` (contact info submitted) | `checkout_contact_info_submitted` | checkout snapshot |
| `checkout_step` (address submitted) | `checkout_address_info_submitted` | checkout snapshot |
| `checkout_step` (shipping selected) | `checkout_shipping_info_submitted` | checkout snapshot |
| `checkout_step` (payment submitted) | `payment_info_submitted` + `checkout_completed` | checkout snapshot |

> **Checkout step mapping note:** Storefront Next fires `checkout_step` on *arrival* at each step, not on submission. The table above reflects this: the Noibu event fires when the user advances to the next step, which confirms the previous step was completed.

### Consent

If `consentCategory` is set on the adapter config, the adapter checks `consentPreferences` before forwarding any event. No `consentCategory` means all events pass through as long as `consentPreferences` is a non-empty array (i.e. the user has accepted at least one category).

```ts
createNoibuAdapter({ consentCategory: 'analytics' })
```

## Development

The extension source lives in two places that must be kept in sync:

| Location | Purpose |
|---|---|
| `sfcc-storefront/src/extensions/noibu/` | Active development and local testing |
| `commerce-apps/analytics/noibu/commerce-noibu-app-v1.0.0/storefront-next/src/extensions/noibu/` | Commerce App package source |

### Workflow

All commands run from the repo root.

**1. Make changes and run tests**

Edit files in `sfcc-storefront/src/extensions/noibu/`, then verify:

```bash
cd sfcc-storefront && pnpm test src/extensions/noibu          # run once
cd sfcc-storefront && pnpm test:watch src/extensions/noibu   # watch mode
```

**2. Sync to the Commerce App package**

```bash
cp -r sfcc-storefront/src/extensions/noibu/. \
  commerce-apps/analytics/noibu/commerce-noibu-app-v1.0.0/storefront-next/src/extensions/noibu/
```

**3. Pull upstream commerce-apps updates (when needed)**

To get the latest changes from `SalesforceCommerceCloud/commerce-apps` into the `Noibu/commerce-apps` fork:

```bash
cd commerce-apps
git pull https://github.com/SalesforceCommerceCloud/commerce-apps.git main
git push origin main
```

This is safe — upstream never touches `analytics/noibu/`, so there are no conflicts.

**4. Package and publish** — see [Packaging and Distribution](#packaging-and-distribution) below.

## Packaging and Distribution

The app is distributed as a Commerce App Package (CAP) — a ZIP file consumed by the Commerce App Registry.

### 1. Validate

Before packaging, run the app validator from the `commerce-apps/` directory:

```
/validate-app
```

All checks must pass.

### 2. Create the ZIP

From `commerce-apps/analytics/noibu/`:

```bash
zip -r noibu-v1.0.0.zip commerce-noibu-app-v1.0.0/ \
  -x "*.DS_Store" -x "__MACOSX/*" -x "*/.*" -x "Thumbs.db"
```

Verify the archive has a single root directory:

```bash
unzip -l noibu-v1.0.0.zip | head -20
```

### 3. Compute the SHA-256 hash

```bash
shasum -a 256 analytics/noibu/noibu-v1.0.0.zip
```

### 4. Update the root manifest

Add or update the entry in `commerce-apps-manifest/manifest.json`:

```json
{
  "id": "noibu",
  "name": "Noibu",
  "description": "Loads Noibu session monitoring script on every storefront page",
  "domain": "analytics",
  "type": "app",
  "provider": "thirdParty",
  "version": "1.0.0",
  "zip": "noibu-v1.0.0.zip",
  "sha256": "<computed hash>"
}
```

### 5. Commit

Only commit:

- `analytics/noibu/noibu-v1.0.0.zip`
- `commerce-apps-manifest/manifest.json`
- `commerce-apps-manifest/translations/*.json`
- `analytics/noibu/catalog.json` (first release only)

Do **not** commit the extracted `commerce-noibu-app-v1.0.0/` directory.

### Version bumps

For a new version, bump `"version"` in `commerce-app.json`, rename the app directory to match (`commerce-noibu-app-v1.1.0/`), re-run the steps above with the new version number, and remove the old ZIP before committing.
