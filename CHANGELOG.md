# Changelog

All notable changes to this project are documented in this file.

## [Unreleased]

### Added

- Repository restructured into `plugin/` (universal, English, currency-only) and `plugin-cz/` (legacy, Czech, currency + language) variants.
- New **universal** plugin (`plugin/crx-geo-currency.php`): fully in English, and configurable for any country/currency combination through a single `crx_glc_config` filter instead of hardcoded values. Scope is intentionally currency-only.
- Currency mapping, default currency, cookie TTL, and the bot-detection pattern are all filterable — no code edits required for typical setups.
- Valid, importable Code Snippets JSON export for the universal plugin (`plugin/crx-geo-currency.code-snippets.json`).
- MIT `LICENSE`, `.gitignore`, and this `CHANGELOG.md`.
- Credited the original author, Matěj Horák, and noted that the original snippet was built for Crystalex (crystalexcz.com).

### Unchanged

- `plugin-cz/` keeps the original Czech/Slovak-only snippet byte-for-byte as it shipped in v2.0 — no logic, naming, or comments were touched. It still does both currency (CZ→CZK / else EUR) and language switching (cs·sk→cs / else en). Use it if you specifically want that behavior out of the box, in Czech.

## [2.0.0] - Legacy CZ plugin
- Fix for currency not being applied correctly on a visitor's very first pageview (see `plugin-cz/`).

## [1.0.0] - Legacy CZ plugin
- Initial release: geolocation-based currency switching (CZ→CZK, else EUR) and browser-language-based language switching (cs/sk→cs, else en) for WooCommerce + WPML/WCML.
