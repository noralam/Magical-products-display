# Magical Shop Builder Audit Findings

Date: 2026-05-01

Scope: first-pass review of the full plugin for errors, security issues, WordPress coding standards, duplicate/mismatched code, and WooCommerce environment compatibility. No code fixes were applied in this pass.

Syntax check: `php -l` passed for all plugin PHP files outside `vendor`.

## High Priority

1. Single-product AJAX add-to-cart has no nonce verification.
   - Files: `includes/functions/woocommerce-functions.php`, `assets/js/widgets/mpd-add-to-cart.js`, `includes/widgets/single-product/class-mpd-widget-add-to-cart.php`
   - Details: `mpd_single_product_add_to_cart()` accepts public `wp_ajax_nopriv_mpd_single_add_to_cart` requests and modifies the cart, but it does not call `check_ajax_referer()` or `wp_verify_nonce()`. The widget wrapper and `mpd-add-to-cart.js` also do not send a nonce for this request.
   - Risk: CSRF/cart manipulation from third-party pages. Also weakens WooCommerce add-to-cart request integrity.

2. Checkout nonce names are inconsistent.
   - Files: `includes/assets-managment.php`, `includes/functions.php`, `includes/widgets/checkout/class-mpd-widget-multi-step-checkout.php`
   - Details: `mpd-checkout-widgets` localizes `wp_create_nonce( 'mpd_checkout_nonce' )`, while checkout AJAX handlers verify `mpd-checkout-nonce`. The multi-step checkout widget separately embeds `wp_create_nonce( 'mpd-checkout-nonce' )` in a data attribute.
   - Risk: Some checkout/cart AJAX flows may fail security checks depending on which script/widget is used, causing broken WooCommerce checkout interactions.

3. Public layout REST API exposes server layout data without authentication.
   - File: `libs/mpd-layout-server/includes/class-mpd-layout-rest-api.php`
   - Details: All layout server REST routes use `public_permission_check()`, which currently returns `true`, including `/layouts`, `/layouts/{id}/structure`, `/verify`, and `/info`. The comment says API key validation should be implemented in production.
   - Risk: Published layout structures and server metadata are publicly accessible. If pro/internal layout data is stored there, it can be scraped.

4. Public AJAX product tab endpoint trusts client-provided widget settings.
   - File: `includes/ajax/products-tab-ajax.php`
   - Details: `mpd_load_tab_products` accepts a full `settings` array from `$_POST`, sanitizes it, and uses it to build product output rather than loading trusted Elementor widget settings from the saved page.
   - Risk: Frontend users can alter product count, display settings, query behavior, and generated output beyond the saved widget configuration.

5. Frontend cart-changing AJAX endpoints are available to unauthenticated users.
   - Files: `includes/functions/woocommerce-functions.php`, `includes/functions.php`
   - Details: `mpd_add_to_cart`, `mpd_single_add_to_cart`, `mpd_refresh_order_review`, and `mpd_update_cart_quantity` are all registered for `nopriv`. Some are expected for WooCommerce carts, but they need consistent nonces, cart/session guards, and behavior parity with WooCommerce native handlers.
   - Risk: Cart/session mutation endpoints are exposed broadly; missing or mismatched nonce checks can become CSRF or reliability issues.

## WooCommerce Compatibility Issues

6. `mpd_single_product_add_to_cart()` can fatal if WooCommerce cart is unavailable.
   - File: `includes/functions/woocommerce-functions.php`
   - Details: The handler calls `WC()->cart->add_to_cart()` and `WC()->cart->get_cart_hash()` without checking `function_exists( 'WC' )`, `WC()`, or `WC()->cart`.
   - Risk: AJAX requests during unusual WooCommerce/session states can throw fatal errors instead of returning JSON errors.

7. Older `mpd_ajax_add_to_cart()` does not support variable product data.
   - File: `includes/functions/woocommerce-functions.php`
   - Details: This handler only reads `product_id` and `quantity`, then calls `WC()->cart->add_to_cart( $product_id, $quantity )`. The newer single-product handler supports variation ID and attributes.
   - Risk: If any widget/script still uses `mpd_add_to_cart`, variable products will fail or add incorrectly.

8. Checkout AJAX manually initializes WooCommerce sessions.
   - File: `includes/functions.php`
   - Details: `mpd_refresh_order_review_ajax()` and `mpd_update_cart_quantity_ajax()` create `new WC_Session_Handler()` when `WC()->session` is missing.
   - Risk: This may not fully match WooCommerce's normal frontend initialization sequence and can behave differently across WooCommerce versions, caching layers, or custom session handlers.

9. Product archive AJAX mutates `$_GET` during requests.
   - File: `includes/ajax/products-archive-ajax.php`
   - Details: AJAX filter/load-more parses a posted query string and writes allowed values into `$_GET` so WooCommerce filters can read them.
   - Risk: This works, but it is fragile and can create mismatches with other hooks expecting the original request state. A local query/filter context would be safer.

10. WooCommerce dependency checks happen before most plugin loading, but some bundled classes assume WooCommerce APIs later.
    - Files: `magical-products-display.php`, `includes/functions/woocommerce-functions.php`, multiple widget files under `includes/widgets/`
    - Details: The main plugin stops initialization if WooCommerce is missing, but direct includes/tests or partial loading can still hit functions that assume WooCommerce functions/classes exist.
    - Risk: Lower risk in normal WP admin/frontend use, but fragile for tests, CLI, and partial includes.

## Security And Escaping Issues

11. AJAX search nonce inputs are sanitized without `wp_unslash()`.
    - File: `includes/widgets/ajax-search/ajax-search-handler.php`
    - Details: `widget_id` and `nonce` are read from `$_POST` with `sanitize_text_field()` directly; later fields use `wp_unslash()` correctly.
    - Risk: WordPress coding standards issue and possible nonce mismatch in slashed input edge cases.

12. Wishlist and compare AJAX HTML concatenates WooCommerce image/price HTML without explicit escaping at concatenation points.
    - File: `includes/functions/woocommerce-functions.php`
    - Details: `$thumbnail` and `$price` from WooCommerce are appended into AJAX response HTML directly. These are usually WooCommerce-generated safe HTML, but the escaping intent is not documented or constrained with `wp_kses_post()`.
    - Risk: Standards/security review noise; safer to explicitly allow expected HTML.

13. Dynamic HTML attributes are sometimes assembled into strings and echoed with PHPCS ignores.
    - Files: multiple widget files, especially checkout/global/archive widgets
    - Details: Several widgets build attribute strings manually and output them with `WordPress.Security.EscapeOutput.OutputNotEscaped` ignores.
    - Risk: Some ignores are justified, but the pattern increases review burden and can hide real escaping gaps if any attribute source changes.

14. Public quick-view endpoint renders product short description through `wp_kses_post()`.
    - File: `includes/functions/woocommerce-functions.php`
    - Details: This is usually acceptable for post content, but quick-view is exposed through public AJAX and returns arbitrary product short description HTML.
    - Risk: Depends on who can edit products and allowed HTML. Worth documenting as intentional or applying stricter sanitization if needed.

## Duplicate Or Mismatched Code

15. Product archive query logic is duplicated between the archive widget and archive AJAX handler.
    - Files: `includes/widgets/shop-archive/class-mpd-widget-products-archive.php`, `includes/ajax/products-archive-ajax.php`
    - Details: Filtering by order, attributes, categories, tags, brand/custom taxonomies, price, rating, stock, sale, and featured appears in both places.
    - Risk: Fixes can be applied to one path and missed in the other, creating frontend/AJAX result mismatches.

16. Product card rendering is duplicated across legacy product widgets and AJAX tab rendering.
    - Files: `includes/ajax/products-tab-ajax.php`, `includes/widgets/products-grid.php`, `includes/widgets/products-tab.php`, `includes/widgets/products-carousel.php`, `includes/widgets/products-list.php`, `includes/widgets/products-awesome-list.php`
    - Details: Image, badge, title, excerpt, price, rating, and cart button markup are repeated with different variable names and style branches.
    - Risk: Styling/escaping/WooCommerce behavior can drift between widgets and AJAX-loaded content.

17. Plugin version is inconsistent between PHP and npm package metadata.
    - Files: `magical-products-display.php`, `package.json`
    - Details: PHP plugin header and `Magical_Shop_Builder::VERSION` are `2.0.1`; `package.json` is `2.0.0`.
    - Risk: Build/release confusion and asset/version cache mismatch during packaging.

18. Plugin branding/naming is mixed across files.
    - Files: `magical-products-display.php`, `composer.json`, `README.md`, namespaces/classes/options throughout `includes/`
    - Details: The product is now “Magical Shop Builder”, but package/text/domain/old class names still use “magical-products-display”, `mgp`, and `mpd` naming.
    - Risk: Not a runtime bug by itself, but it increases maintenance confusion and migration mistakes.

19. Pro feature controls are registered in the free widget UI even after a pro notice.
    - File: `includes/widgets/single-product/class-mpd-widget-add-to-cart.php`
    - Details: The Add To Cart widget adds the pro notice when not pro, but still registers quantity style, sticky cart, buy now, icon, and custom button controls. Runtime checks prevent some behavior, but the UI may expose unavailable settings.
    - Risk: User confusion and saved settings that appear configurable but do nothing in free mode.

## WordPress Coding Standards Issues

20. Coding style is inconsistent between new namespaced classes and legacy classes.
    - Files: `includes/admin/*.php`, `includes/templates/*.php`, `includes/ajax/*.php`, legacy widget files under `includes/widgets/`
    - Details: New code mostly uses WordPress spacing/array syntax; older AJAX and widget files use compact `if(...)`, short arrays, inconsistent indentation, and mixed docblock style.
    - Risk: Harder review/maintenance and more PHPCS failures if WordPress standards are enforced.

21. No PHPCS/WordPressCS configuration is present.
    - Files: `composer.json`, repository root
    - Details: `composer.json` only includes PHPUnit polyfills in dev dependencies; no WordPress Coding Standards tooling or ruleset is configured.
    - Risk: Standards regressions are not automatically caught.

22. Several direct superglobal reads need nonce-ignore comments or normalization consistency.
    - Files: `includes/templates/class-mpd-template-manager.php`, `includes/templates/views/single-mpd_template.php`, `includes/widgets/shop-archive/*.php`, `includes/widgets/thankyou/*.php`, `includes/widgets/my-account/*.php`
    - Details: Many frontend display reads from `$_GET` are sanitized, but PHPCS would still flag missing nonce verification for request reads unless intentionally ignored and documented.
    - Risk: Standards noise; real issues can be harder to spot among expected frontend query-param reads.

23. Translation strings sometimes include extra spacing or inconsistent wording.
    - File: `magical-products-display.php`
    - Details: Admin dependency notices include strings such as “currently NOT RUNNING  %3$s” with double spaces and hard-coded style snippets in generated links.
    - Risk: Minor polish/translation quality issue.

## Lower Priority / Follow-Up Checks

24. Need runtime testing against WooCommerce pages.
    - Suggested pages: single product simple/variable, cart, checkout, shop archive filters, tab products, quick view, wishlist/compare/header cart.
    - Reason: Static review found likely issues, but WooCommerce behavior depends heavily on page context, session state, theme hooks, and enabled extensions.

25. Need WordPressCS scan after adding PHPCS tooling.
    - Reason: Manual review found style and escaping patterns, but a full standards report should be generated with a configured ruleset before fixing.
