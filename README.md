# 🌍 WooCommerce Geo Currency

> **CRX Geo Currency** — a WordPress snippet/plugin that switches WooCommerce currency based on visitor geolocation.

Automatically switches **currency by IP geolocation** on WooCommerce stores running [WPML](https://wpml.org/) + [WooCommerce Multilingual (WCML)](https://woocommerce.com/products/woocommerce-multilingual/).

This repository ships **two variants** of the snippet:

| Variant | Folder | Language of code/comments | Scope |
|---|---|---|---|
| **Universal** (recommended) | [`plugin/`](plugin/) | English | Currency only — configurable for any country/currency via one filter |
| **Legacy CZ** | [`plugin-cz/`](plugin-cz/) | Czech | Currency (CZ→CZK / else EUR, hardcoded) **+** language switching (cs·sk→cs / else en, hardcoded) |

If you only need currency switching, use the **universal** variant — it needs no code edits to adapt to a different set of countries/currencies. The legacy CZ variant additionally does browser-language-based language switching and is kept byte-for-byte unchanged for existing installs that depend on it.

---

## 📋 Contents

- [✨ What it does](#-what-it-does)
- [📦 Requirements](#-requirements)
- [🚀 Installation](#-installation)
- [⚙️ Configuring the universal variant](#️-configuring-the-universal-variant)
- [⚙️ How it works — currency](#️-how-it-works--currency)
- [🗣️ Language switching (legacy CZ variant only)](#️-language-switching-legacy-cz-variant-only)
- [📐 Code structure](#-code-structure)
- [💡 Tips & troubleshooting](#-tips--troubleshooting)
- [👤 Author & credits](#-author--credits)
- [📄 License](#-license)

---

## ✨ What it does

| 🎯 Feature | 📝 Behavior | Universal | Legacy CZ |
|---|---|---|---|
| **Currency** | Resolved from the visitor's IP country on every request, via a country→currency map (with a fallback currency for everything else). | ✅ Configurable for any country/currency | ✅ Hardcoded: CZ→CZK, else EUR |
| **Language** | On a visitor's first pageview, detected from the browser and redirected to the matching WPML language; a manual WPML-switcher choice always wins afterwards. | ❌ Not included | ✅ Hardcoded: cs·sk→cs, else en |

---

## 📦 Requirements

- ✅ **WordPress** 5.0+
- ✅ **WooCommerce** (for `WC_Geolocation`)
- ✅ **WPML** + **WooCommerce Multilingual / WCML** (for the `wcml_client_currency` filter)
- ✅ **PHP** 7.4+
- ✅ A way to run the PHP snippet — the [Code Snippets](https://wordpress.org/plugins/code-snippets/) plugin, a standalone plugin file, or an mu-plugin

The legacy CZ variant additionally needs at least two active WPML languages (`cs`/`en`) for its language-switching part.

---

## 🚀 Installation

Pick **one** variant and **one** delivery method.

### Option A — Universal variant (recommended)

#### As a plugin file

1. Upload [`plugin/crx-geo-currency.php`](plugin/crx-geo-currency.php) to `wp-content/plugins/crx-geo-currency/`
2. WordPress admin → **Plugins** → activate **CRX Geo Currency (Universal)**
3. Add your configuration (see [below](#️-configuring-the-universal-variant)) to your theme's `functions.php` or a small site-specific plugin

#### Via the Code Snippets plugin

1. WordPress admin → **Code Snippets** → **Import**
2. Upload [`plugin/crx-geo-currency.code-snippets.json`](plugin/crx-geo-currency.code-snippets.json)
3. Activate the imported snippet
4. Add a second snippet with your `crx_glc_config` filter (see below), scope **Run everywhere**

### Option B — Legacy CZ variant

#### As a plugin file (CZ variant)

1. Upload [`plugin-cz/crx-geo-lang-currency.php`](plugin-cz/crx-geo-lang-currency.php) to `wp-content/plugins/`
2. WordPress admin → **Plugins** → activate **CRX Geo Lang Currency**

#### Via the Code Snippets plugin (CZ variant)

1. WordPress admin → **Code Snippets** → **Add New**
2. Paste the full PHP code from [`plugin-cz/crx-geo-lang-currency.php`](plugin-cz/crx-geo-lang-currency.php) (without the `<?php` tag)
3. **Scope:** Run everywhere
4. Save and activate

> ⚠️ Only run **one** variant at a time — both hook into `wcml_client_currency` and `wp_footer` for currency, and running both would apply the currency logic twice.

---

## ⚙️ Configuring the universal variant

Everything is controlled through a single `crx_glc_config` filter — you never need to edit `plugin/crx-geo-currency.php` itself. Add this to your theme's `functions.php`, a site-specific plugin, or a second Code Snippets snippet:

```php
add_filter( 'crx_glc_config', function ( array $config ): array {

    // Country (ISO 3166-1 alpha-2) => currency (ISO 4217).
    $config['currency_map'] = [
        'CZ' => 'CZK',
        'GB' => 'GBP',
        'US' => 'USD',
    ];

    // Currency for every country not listed above.
    $config['default_currency'] = 'EUR';

    // How long the currency preference cookies last.
    $config['cookie_ttl'] = DAY_IN_SECONDS * 30;

    return $config;
} );
```

Every key is optional — anything you don't override keeps its default:

| Key | Default | Purpose |
|---|---|---|
| `currency_map` | `[]` (empty) | Country → currency overrides |
| `default_currency` | the shop's base currency | Currency for unlisted countries |
| `cookie_ttl` | 30 days | Lifetime of the currency preference cookies |
| `bot_pattern` | a broad bot/crawler regex | Requests matching it skip geolocation entirely |

With no configuration at all, currency switching is a no-op — everyone gets the shop's default currency — so a working setup always needs at least a `currency_map`.

---

## ⚙️ How it works — currency

### ❌ The problem this snippet solves

WCML resolves the currency for every HTTP request like this:

1. On the `init` hook, it reads the currency from cookies (`wcml_client_currency`, `woocommerce_current_currency`)
2. If no cookie exists, it falls back to the shop's default currency
3. It registers price filters (`woocommerce_product_get_price` etc.) as PHP closures that capture that currency value
4. Every price conversion from then on goes through those filters

**Consequence:** if the currency changes *after* the filters are registered, the filters don't notice — they already captured the old currency in step 3's closure.

#### 🔴 First-visit scenario (without the fix)

1. A foreign customer lands on the site (no cookies yet)
2. `init`: WCML reads cookies → none found → sets CZK
3. WCML registers price filters with `currency = 'CZK'` (baked into the closure)
4. `wp` hook: our code calls `set_client_currency('EUR')`
   - → updates the object property, but the closures in the filters still hold `'CZK'`
5. The page renders in EUR (`set_client_currency` does flip the display), but...
6. The customer clicks "Add to cart" (a `wc-ajax` request)
7. WooCommerce reads the price through the filters → closure says CZK → a 600 CZK product is added as 600, unconverted
8. **The cart shows 600 EUR** ← 🐛 **bug**

On the second and later visits the cookies exist (set by JS in the footer), so WCML reads EUR from the cookie, the closure gets the right currency, and everything works.

### ✅ The fix: the `wcml_client_currency` filter

WCML applies the `wcml_client_currency` filter right before it commits to a currency. Hooking into it lets us influence the currency **before** the price filters are registered — at the correct time.

```php
add_filter( 'wcml_client_currency', function ( $currency ) {
    if ( is_admin() ) return $currency;
    if ( crx_glc_is_bot_request() ) return $currency;

    $country = crx_glc_country();
    if ( $country === '' ) return $currency;

    return crx_glc_target_currency( $country );
}, 20 );
```

This filter:

- ✅ Works for every request type — regular pages *and* `wc-ajax` add-to-cart requests
- ✅ Works on the very first visit, with no cookies present
- ✅ Doesn't need cookies at all — it decides purely from the IP

The `add_action('wp', ...)` hook stays as a safety net, keeping the JS cookies in `wp_footer` in sync in case something reads them directly instead of going through the WCML filter.

#### 🟢 Currency flow after the fix

1. A foreign customer lands on the site (no cookies yet)
2. `init`: WCML calls `get_client_currency()`
   - → applies the `wcml_client_currency` filter
   - → our filter returns `'EUR'` based on geo-IP
   - → WCML registers price filters with `currency = 'EUR'` ✓
3. `wp_footer`: JS sets the `wcml_client_currency=EUR` cookie
4. The customer clicks "Add to cart"
5. WooCommerce reads the price → filters convert CZK→EUR ✓
6. The cart shows the correct EUR price ✓

### 🔧 Why `set_client_currency()` alone isn't enough

In short: `set_client_currency()` is called too late — the price filters have already closed over the old currency by the time it runs. See the scenario above for the full walkthrough.

---

## 🗣️ Language switching (legacy CZ variant only)

The **universal** variant in `plugin/` is currency-only. Browser-language-based language switching only exists in the **legacy CZ** variant (`plugin-cz/`), unchanged from the original snippet:

- Runs entirely in JavaScript in `wp_footer`, because only the browser has access to `navigator.languages`
- On a visitor's first pageview (no `crx_manual_lang` cookie yet), detects the browser language: `cs`/`sk` → Czech, everything else → English, then redirects via the matching `hreflang` link (falling back to a `/cs/`↔`/en/` URL-segment swap)
- Once a visitor uses the WPML switcher, the chosen language is written to the cookie and detection never runs again

If you need language switching for a different set of languages, either adapt `plugin-cz/crx-geo-lang-currency.php` directly, or ask for it to be added back to the universal variant as a configurable feature.

---

## 📐 Code structure

```text
defined('ABSPATH') || exit                    ← security guard

CONFIG
└── crx_glc_config()                          ← single source of truth, filterable via `crx_glc_config`

HELPERS
├── crx_glc_is_bot_request()                  ← bot/crawler detection
├── crx_glc_country()                         ← WC_Geolocation, static cache
├── crx_glc_target_currency()                 ← country → currency, via config
└── crx_glc_get_cookie()                      ← $_COOKIE wrapper

JS COOKIE HELPERS
├── crx_glc_output_js_cookie_helpers()        ← prints crxGlcSetCookie/crxGlcGetCookie (printed once)
├── crx_glc_set_wcml_runtime_currency()       ← PHP: calls WCML's set_client_currency()
└── crx_glc_force_geo_currency()              ← PHP: sets currency + schedules JS cookies in wp_footer (priority 5)

HOOKS
├── add_filter('wcml_client_currency', ..., 20)  ← PRIMARY currency fix
└── add_action('wp', ..., 10)                    ← safety net + JS cookies
```

---

## 💡 Tips & troubleshooting

- **Testing**: use DevTools to inspect the `wcml_client_currency` and `woocommerce_current_currency` cookies
- **Debugging**: check the browser console and Network tab for JS errors
- **Performance**: `crx_glc_country()` uses a static cache, so it's cheap even on high-traffic sites
- **Currency doesn't change**: confirm WooCommerce Multilingual (WCML) is active — the `wcml_client_currency` filter only exists when WCML is loaded
- **Bots see the shop's default currency on purpose**: that's intentional — the `bot_pattern` filter keeps crawlers on the shop's default currency so cached/indexed content stays consistent. Override it via `crx_glc_config` if it's too broad or too narrow for your traffic

---

## 🐛 Reporting bugs

Found a bug or unexpected behavior? Open an [issue on GitHub](https://github.com/Braska-botmaker/woocomerce-geo-lang-currency/issues). To make it fixable, please include:

- Which variant you're using (`plugin/` universal or `plugin-cz/` legacy)
- WordPress, WooCommerce, WPML and WCML versions
- Your `crx_glc_config` filter, if you've customized it
- Steps to reproduce (ideally including the country/IP or browser language involved) and what you expected vs. what happened

---

## 👤 Author & credits

Created and maintained by **Matěj Horák**.

The original snippet (the [`plugin-cz/`](plugin-cz/) variant) was built for [Crystalex](https://crystalexcz.com). This repository generalizes it into a reusable, universal plugin while keeping that original version intact.

---

## 📄 License

MIT — see [LICENSE](LICENSE). Use and modify freely.
