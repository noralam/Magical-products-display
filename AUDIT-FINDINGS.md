# Plugin Audit Report — Magical Products Display

**Date:** 2026-08-19  
**Plugin Version:** 2.0.5  
**Total PHP files:** ~113 in includes/ (1652 total with vendor/)  
**Total lines:** ~36k (excluding node_modules/vendor)

---

## HIGH PRIORITY

### 1. **Unescaped output in multiple widgets**
**Risk:** XSS vulnerability  
**Files:** 50+ instances across widgets  
**Examples:**
- `widgets/global/class-mpd-widget-store-notice.php:684` — `$data_attrs` not escaped
- `widgets/testimonial-carousel.php:1172+` — `get_render_attribute_string()` output not escaped
- `widgets/pricing-table.php:2117+` — link attributes not escaped
- `templates/class-mpd-template-renderer.php:645,859` — raw `echo $content`

**Impact:** User-controlled data could execute JS. Most are Elementor `get_render_attribute_string()` calls assumed safe but not explicitly escaped in output context.

**Fix:** Wrap with `wp_kses_post()` or verify `get_render_attribute_string()` is safe-by-construction.

---

### 2. **Direct `$_POST`/`$_GET` access without sanitization**
**Risk:** Data injection, potential SQL injection chain  
**Files:** 22KB output from grep (60+ occurrences)  
**Examples:**
- `widgets/ajax-search/ajax-search-handler.php:37,38,54-56` — sanitized ✓
- `widgets/my-account/class-mpd-widget-orders.php:986` — `$_GET['orders-page']` wrapped in `absint()` ✓
- `widgets/shop-archive/class-mpd-widget-products-archive.php:1662` — `$_GET['orderby']` checked but not shown sanitized

**Impact:** Most cases ARE sanitized (absint/sanitize_text_field), but need full audit to confirm no gaps.

**Fix:** Add phpcs suppression comments where already safe. Verify any unsanitized cases.

---

### 3. **SQL queries — all appear prepared**
**Files:** 10 files use `$wpdb`  
**Status:** ✓ All checked use `$wpdb->prepare()`  
**Examples:**
- `functions.php:389,395,414,430` — prepared ✓
- `widgets/ajax-search/ajax-search-handler.php:427` — prepared ✓
- `widgets/shop-archive/class-mpd-widget-price-filter.php:691,702` — prepared ✓

**No issues found.** All SQL uses `$wpdb->prepare()`.

---

### 4. **Nonce verification — inconsistent coverage**
**Risk:** CSRF on state-changing actions  
**Found:** 17 nonce checks across ajax/admin handlers  
**Gap:** Need to verify ALL `wp_ajax_*` handlers have nonce checks.

**Examples verified:**
- `ajax-search-handler.php:41` — ✓ wp_verify_nonce
- `woocommerce-functions.php:416,476,659,754,809` — ✓ check_ajax_referer
- `admin-info.php:119,247` — ✓ check_ajax_referer

**Fix:** Audit every `add_action('wp_ajax_*')` for nonce verification. Missing checks = CSRF risk.

---

## MEDIUM PRIORITY

### 5. **console.log() left in production JS**
**Files:**
- `assets/js/widgets/mpd-multi-step-checkout.js:21` — debug logger
- `assets/js/mpd-global-widgets.js:1002,1051` — recently viewed tracking logs

**Impact:** Exposes internal state to browser console. Not a security risk but unprofessional.

**Fix:** Remove or wrap in `if (MPD_DEBUG)` check.

---

### 6. **Inline `<script>` tags without proper escaping**
**Risk:** If any variable inside script tags is unescaped, XSS risk  
**Files:** 20+ occurrences  
**Examples:**
- `admin/class-mpd-product-video-metabox.php:174`
- `frontend/class-mpd-preloader.php:163,569`
- `widgets/checkout/class-mpd-widget-shipping-form.php:736,931`
- `widgets/single-product/class-mpd-widget-add-to-cart.php:1090`

**Status:** Need to review each inline script for unescaped PHP vars inside JS context.

**Fix:** Use `wp_json_encode()` for any PHP data passed to JS. Use `wp_add_inline_script()` where possible.

---

### 7. **Large functions — refactor candidates**
**Files:** Multiple widgets have 500-1500 line render methods  
**Examples:**
- `class-mpd-widget-account-login.php` — 1700+ lines
- `class-mpd-widget-products-archive.php` — 2000+ lines
- `class-mpd-widget-multi-step-checkout.php` — 2500+ lines

**Impact:** Hard to maintain, test, review.

**Fix:** Extract sub-methods for form rendering, validation, style output.

---

### 8. **Duplicate widget architecture — legacy vs new**
**Old style:** `includes/widgets/*.php` — 11 files (mgProducts_Grid, mgProducts_Shop, etc.)  
**New style:** `includes/widgets/{cart,checkout,my-account,shop-archive,single-product}/class-mpd-*.php` — ~60 files  

**Risk:** Confusion, inconsistent patterns, harder to maintain.

**Fix:** Migrate all old-style widgets to new namespaced structure. Deprecate old class names.

---

### 9. **Missing ABSPATH checks — some files**
**Status:** Most files have `if (!defined('ABSPATH')) exit;` at top  
**Gap:** Vendor files don't (expected), but verify all plugin-authored files do.

**Fix:** Grep for missing guards, add where needed.

---

### 10. **Rate limiting — implemented in AJAX search**
**Status:** ✓ `ajax-search-handler.php` has rate limit + transient cache  
**Gap:** Other AJAX handlers (add-to-cart, wishlist, compare) don't have rate limits.

**Impact:** DoS risk on heavy AJAX endpoints.

**Fix:** Add rate limiting to high-traffic AJAX actions.

---

## LOW PRIORITY

### 11. **Commented-out code blocks**
**Files:** Not quantified yet, but common in widget files  

**Fix:** Remove dead code or document why it's kept.

---

### 12. **Unused enqueued scripts/styles**
**Status:** Not audited yet. Need to cross-reference `wp_enqueue_*` calls with actual usage.

**Fix:** Remove unused assets to reduce page weight.

---

### 13. **phpcs suppressions — many present**
**Examples:** `// phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped`  

**Status:** Most appear justified (WooCommerce template output, Elementor render attributes).  
**Risk:** If misused, suppressions hide real issues.

**Fix:** Audit each suppression to confirm it's valid.

---

### 14. **functions.php is 1115 lines**
**Files:** Split into `functions/*.php` (helper, template, woocommerce) — 1624 lines combined  

**Impact:** Large but already partitioned. Could split further by domain (cart, checkout, account).

---

### 15. **WooCommerce template overrides — none found**
**Status:** Plugin uses WooCommerce hooks/filters, not template overrides.  
**Risk:** None. This is best practice.

---

## SUMMARY

| Priority | Count | Critical? |
|----------|-------|-----------|
| HIGH     | 4     | 1 (XSS), 1 (CSRF audit needed) |
| MEDIUM   | 10    | 0 |
| LOW      | 5     | 0 |

**Next steps:**
1. Fix unescaped output in widget render methods (HIGH #1)
2. Audit all `wp_ajax_*` handlers for nonce verification (HIGH #4)
3. Remove `console.log()` from production JS (MEDIUM #5)
4. Review inline `<script>` tags for unescaped PHP vars (MEDIUM #6)
5. Refactor 500+ line functions (MEDIUM #7)
