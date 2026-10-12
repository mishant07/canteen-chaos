# Debug Log

---

### CC-01 — "The search suggestions are behind everything"

**Reproduced:**
Opened the menu page on desktop, typed "ch" into the search box to trigger suggestions. The suggestion dropdown rendered physically underneath the horizontal category bar (`.cat-tabs`), making all suggestions except the very top edge unreachable and unclickable.

**Cause:**
Stacking context hierarchy issue in `frontend/style.css`. `.search-wrap` was declared with `position: relative; z-index: 1;`, whereas the adjacent `.cat-tabs` bar directly below was declared with `position: relative; z-index: 40;`. Because both elements share the `.filters` container stacking context and `.cat-tabs` had a higher z-index (40 > 1) and appears later in the DOM, `.cat-tabs` painted on top of the `.suggest-box` dropdown.

**Fix:**
In `frontend/style.css` (line 756), increased `.search-wrap` to `z-index: 50;`. This elevates the search container and its child `.suggest-box` above `.cat-tabs` (z-index 40) so the suggestions overlay seamlessly.

**Checked:**
Typed various queries into the search bar. All suggestions appeared fully visible above the category tabs, hovered with background highlighting, and were immediately clickable.

**Time:**
15 minutes.

---

### CC-02 — "Can't read anything in dark mode"

**Reproduced:**
Toggled the dark mode switch (`html[data-theme="dark"]`). Dish cards displayed dark backgrounds (`#201913`), but the dish names and dish prices were rendered in a dark brownish-black color (`#2b2118`), making them virtually illegible against the card background.

**Cause:**
Hardcoded color in `frontend/style.css` at line 294: `.dish-body` had `color: #2b2118;`. While the theme defines `--card: #201913;` and `--ink: #f2e8df;` for dark mode, the hardcoded hex color overrode the dynamic CSS variable inheritance.

**Fix:**
Replaced `color: #2b2118;` with `color: var(--ink);` on `.dish-body` in `frontend/style.css`. In light mode, `--ink` resolves to `#2b2118`, and in dark mode, it resolves to `#f2e8df`.

**Checked:**
Switched back and forth between light and dark modes. Dish titles and prices adapt cleanly to crisp, high-contrast light text (`#f2e8df`) in dark mode, while retaining the original dark ink styling in light mode.

**Time:**
10 minutes.

---

### CC-03 — "The menu is wider than my phone"

**Reproduced:**
Simulated mobile viewport widths (375px–760px). The menu grid forced horizontal scrolling, and the "Add to Cart" button along with the right edge of cards in the right-hand column overflowed beyond the screen boundary.

**Cause:**
Rigid dimensions and unyielding track constraints in `frontend/style.css` under `@media (max-width: 760px)`:
1. `.name-btn` was styled with `width: max-content;`, preventing dish titles from wrapping and forcing card containers to widen to fit the longest dish name.
2. `.dish-card img` had `min-width: 100%; height: 170px; aspect-ratio: 3 / 2;`, which mechanically enforces an image width of `170px * 1.5 = 255px`. Two columns of 255px plus grid gap and padding exceed mobile screen widths.
3. Grid columns were defined as `repeat(2, 1fr)`. In CSS Grid, `1fr` defaults to `minmax(auto, 1fr)`, allowing tracks to expand past the viewport when content cannot shrink.

**Fix:**
In `frontend/style.css`:
- Updated `.name-btn` to `width: 100%; text-align: left;` to allow multi-line title wrapping.
- Replaced the rigid `aspect-ratio: 3 / 2` and `min-width: 100%` on `.dish-card img` with `width: 100%; height: 170px; object-fit: cover;`.
- Configured `.menu-grid` columns with `repeat(2, minmax(0, 1fr))` so tracks shrink smoothly without pushing siblings off-screen.

**Checked:**
Tested viewports from 320px to 760px. Dish cards wrap and align inside the screen with no horizontal scrolling or clipped buttons.

**Time:**
25 minutes.

---

### CC-04 — "The buttons don't work on my tablet"

**Reproduced:**
Set browser viewport between 761px and 900px (standard tablet portrait size). Clicked "Add to Cart" and the favorite star button on dish cards; neither button triggered any response, despite working normally at widths < 760px and > 900px.

**Cause:**
In `frontend/style.css` lines 1032–1047, a responsive block for `@media (min-width: 761px) and (max-width: 900px)` generated invisible pseudo-elements `.dish-card::after` (height 58px, z-index 3 at the bottom of the card) and `.img-wrap::after` (width 52px, height 52px, z-index 1 at the top-right). These overlays lacked `pointer-events: none;`, thereby intercepting all mouse/touch click events meant for the Add to Cart button and Favorite button.

**Fix:**
Added `pointer-events: none;` to `.dish-card::after, .img-wrap::after` in `frontend/style.css` so touch/pointer events pass directly through to the interactive buttons underneath.

**Checked:**
Resized viewport to 768px and 820px (iPad/tablet dimensions). Both "Add to Cart" and favorite toggle buttons now click and update state immediately.

**Time:**
20 minutes.

---

### CC-05 — "The category bar scrolls away on my phone"

**Reproduced:**
On viewports <= 480px, scrolled down the menu page. The category tabs and filter bar scrolled off the top of the viewport instead of sticking to the top.

**Cause:**
In `frontend/style.css` line 703, the media query `@media (max-width: 480px)` declared `.view { overflow-x: hidden; }`. In CSS layout rules, applying `overflow-x: hidden` (or any non-visible overflow) to an ancestor container establishes a scroll/clipping context that disables `position: sticky` on child elements (`.filters` is inside `.view`).

**Fix:**
Removed `.view { overflow-x: hidden; }` from the mobile media query. Since the horizontal layout overflow was solved at the root cause in CC-03, clipping on `.view` was neither needed nor correct.

**Checked:**
Tested scrolling on mobile dimensions (375px, 414px, 480px). The category and filter bar remains firmly pinned to the top of the screen as the user scrolls through the dishes.

**Time:**
15 minutes.

---

### CC-06 — "I ordered more than they had"

**Reproduced:**
Sent an order with a dish quantity greater than available stock (e.g. dish had stock: 2, order requested qty: 5). The order was accepted with HTTP 201, and the inventory stock was decremented to negative numbers (`-3`).

**Cause:**
1. In `backend/logic/validation.js`, `validateLine()` checked `if (Number(dish.stock) <= 0)` for sold-out status, but never verified whether `qty > dish.stock`.
2. In `reserveStock()`, the function decremented `dish.stock -= item.qty` without confirming `dish.stock >= item.qty`.

**Fix:**
In `backend/logic/validation.js`:
- Added a validation rule in `validateLine()`:
  `else if (qty > Number(dish.stock)) errors.push(`${dish.name}: only ${dish.stock} left in stock`);`
- In `reserveStock()`, checked `if (dish.stock < item.qty)` and pushed an error to abort reservation if stock was insufficient.

**Checked:**
Sent an order with `qty: 99` on an item with low stock. The API rejected the order with HTTP 400 Bad Request ("only X left in stock"). Verified via `smoke.js` (`qty over stock rejected`).

**Time:**
20 minutes.

---

### CC-07 — "Cancelling makes it worse"

**Reproduced:**
Placed an order for 1 dish (stock dropped from 25 to 24). Subsequently sent a cancellation request (`PATCH /api/orders/:token/status` with `status: 'cancelled'`). Instead of restoring stock to 25, the stock decreased further to 23.

**Cause:**
Sign error in `backend/logic/validation.js` inside `releaseStock()` (line 123):
```javascript
return { ...dish, stock: dish.stock - line.qty };
```
It was subtracting `line.qty` instead of adding it back to `dish.stock`.

**Fix:**
Changed `dish.stock - line.qty` to `dish.stock + line.qty` in `backend/logic/validation.js`.

**Checked:**
Created an order and cancelled it via API. Stock increased back to its original pre-order value. Verified with `smoke.js` (`cancel puts stock back`).

**Time:**
10 minutes.

---

### CC-08 — "An old coupon still works"

**Reproduced:**
Requested a cart quote with coupon code `FRESHERS24` (which had an expiration timestamp of `2024-09-30T23:59:59.000Z` in `backend/data/coupons.json`). The server successfully returned a quote applying a Rs. 30 discount.

**Cause:**
In `backend/logic/pricing.js`, `applyCoupon()` checked usage limits (`usesLeft <= 0`), minimum order totals (`subtotal < coupon.minOrder`), vegetarian restrictions (`coupon.vegOnly`), and meal slot eligibility (`coupon.slots`), but completely omitted a check for `coupon.expiresAt`.

**Fix:**
In `backend/logic/pricing.js` (line 89), added an expiration check:
```javascript
if (coupon.expiresAt && new Date(coupon.expiresAt).getTime() <= now.getTime()) {
  return { valid: false, discount: 0, reason: 'This coupon has expired' };
}
```

**Checked:**
Sent quote requests with `FRESHERS24`. The discount was refused with `couponError: 'This coupon has expired'`. Verified via `smoke.js` (`expired coupon refused`).

**Time:**
15 minutes.

---

### CC-09 — "The menu shows more dishes than it should"

**Reproduced:**
Requested `GET /api/menu?limit=5&page=1`. Instead of receiving 5 items, the response returned all dishes in the database at once.

**Cause:**
In `backend/logic/search.js`, `paginate()` computed the correct page slice:
```javascript
const items = list.slice(start, start + perPage);
```
However, the returned object incorrectly had `items: list` (the entire un-sliced array) instead of `items: items`.

**Fix:**
In `backend/logic/search.js` line 101, changed `items: list` to `items,`.

**Checked:**
Tested pagination requests. Page 1 returned exactly the specified limit (5 items), and page 2 returned the next page of items. Verified via `smoke.js` (`pages differ`).

**Time:**
10 minutes.

---

### CC-10 — "Sorting by price is backwards"

**Reproduced:**
Selected "Price: low to high" (`sort=price-asc`) and received dishes ordered from most expensive to cheapest. Selected "Price: high to low" (`sort=price-desc`) and received dishes ordered from cheapest to most expensive.

**Cause:**
In `backend/logic/search.js` lines 79–80, comparator implementations in `SORTERS` were reversed:
```javascript
'price-asc': (a, b) => b.price - a.price,
'price-desc': (a, b) => a.price - b.price,
```
`b.price - a.price` sorts in descending order (highest first), while `a.price - b.price` sorts ascending.

**Fix:**
Swapped the expressions in `backend/logic/search.js`:
```javascript
'price-asc': (a, b) => a.price - b.price,
'price-desc': (a, b) => b.price - a.price,
```

**Checked:**
Queried `/api/menu?sort=price-asc` and `/api/menu?sort=price-desc`. Verified that prices monotonically increase in `price-asc` and monotonically decrease in `price-desc`. Verified via `smoke.js` (`sort by price works`).

**Time:**
10 minutes.

---

## Could not fix / Engineering Reflections

### Concurrency Race Condition in File-Based Storage
- **Observation:** In `backend/routes/orders.routes.js`, stock is checked via `validation.validateOrder()` and reserved via `validation.reserveStock()`, after which `store.writeMenu(reserved.menu)` writes synchronously to `menu.json`. However, because Express processes concurrent asynchronous HTTP requests across multiple event ticks (such as price calculations and async middleware), two incoming requests could read identical stock levels concurrently before either write finishes.
- **Why it was left in place:** Solving this properly requires either an in-memory lock/mutex queue per dish ID or an ACID-compliant transactional database (e.g. SQLite / PostgreSQL with row locking). Implementing an external database would exceed repository scope and break the JSON file architecture.

---

## Extra credit

### Unreported Bug 1: Coupon `usesLeft` Was Never Decremented
- **How I noticed it:** While inspecting `backend/routes/orders.routes.js` (lines 157–161) to trace coupon validation for CC-08, I noticed the coupon update block after an order is placed:
  ```javascript
  if (bill.couponCode) {
    store.update('coupons', (list) =>
      list.map((c) => (c.code === bill.couponCode ? { ...c, usesLeft: c.usesLeft } : c))
    );
  }
  ```
  `usesLeft: c.usesLeft` left the count unchanged! This meant coupons with limited uses could be redeemed infinite times.
- **Fix:** Changed the update to `usesLeft: Math.max(0, c.usesLeft - 1)`.

### Unreported Bug 2: Filter by `memberId` Broke Due to Undefined Variable
- **How I noticed it:** In `backend/routes/orders.routes.js` line 187:
  ```javascript
  if (req.query.memberId) orders = orders.filter((o) => o.memberId === req.query.userId);
  ```
  The parameter was named `memberId` in the query check, but filtered against `req.query.userId` (which is `undefined`). Thus, any call filtering orders by member ID always returned an empty array.
- **Fix:** Corrected line 187 to `o.memberId === req.query.memberId`.

### Unreported Bug 3: Orders in 'ready' Status Could Never Be Marked as 'picked'
- **How I noticed it:** In `backend/logic/status.js`, line 6 included `'ready'` inside the `TERMINAL` array:
  ```javascript
  const TERMINAL = ['ready', 'picked', 'cancelled', 'refunded'];
  ```
  Because `'ready'` was classified as terminal, `allowedNext('ready')` returned `[]` and `canTransition('ready', 'picked')` threw an error stating *"A ready order cannot change"*. This prevented canteen staff from marking ready orders as picked up.
- **Fix:** Removed `'ready'` from `TERMINAL` so `allowedNext('ready')` returns `['picked']`.

### Regression Verification
- Executed the full automated test suite `node smoke.js` against the running server. All **90 test assertions passed with 0 failures**.
