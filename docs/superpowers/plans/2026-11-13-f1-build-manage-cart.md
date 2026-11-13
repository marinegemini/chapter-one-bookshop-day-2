# F1 — Build and Manage a Cart Implementation Plan

**Status:** Approved
**Plan approval:** Approved by Workspace user via chat on 2026-11-13
**Spec version:** Approved design spec, 2026-11-13
**Spec:** `docs/superpowers/specs/2026-11-13-f1-build-manage-cart-design.md`
**Intent:** `docs/intent.md`, F1 — Build and manage a cart, approved 2026-11-13
**Intended artifact path:** `docs/superpowers/plans/2026-11-13-f1-build-manage-cart.md`
**Final review path:** `docs/superpowers/reviews/f1-build-manage-cart-review.md`

> **For agentic workers:** REQUIRED SUB-SKILL: Use `subagent-driven-development` or `executing-plans` to implement this plan task by task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an anonymous shopper create an isolated Cart, add available Books without duplicate Line items, view all Line items together, change quantities within current Stock, and remove Line items.

**Architecture:** Extend the in-memory Store with server-managed Carts identified by opaque UUIDs. Add an Express Cart router that validates requests at the HTTP edge and returns current Book data with each Line item. Extend the vanilla JavaScript storefront with a dedicated Cart view, same-origin API calls, `localStorage` of only the `cartId`, pure render helpers, and guarded DOM bootstrap.

**Tech stack:** Node.js 20+, Express 4, TypeScript 5, vanilla JavaScript, HTML, CSS, Vitest, Supertest.

## Global Constraints

- Do not add dependencies, frameworks, build tooling, databases, authentication, sessions, payment handling, or persistent server storage.
- The Store remains the in-memory source of truth; Cart mutations do not reserve or reduce Stock.
- Routes validate request bodies before calling the Store.
- API failures use `{ "error": "message" }`.
- Local TypeScript imports end in `.js`.
- Frontend requests use same-origin `fetch`.
- Only the opaque `cartId` is stored in `localStorage`.
- All styling remains in `public/styles.css`.
- Render helpers remain pure and exported.
- Escape every Book field or API error inserted into HTML.
- DOM bootstrap remains guarded for Vitest imports.
- Mutating API tests reset the Store in `beforeEach`.
- Tests assert external HTTP and rendered behavior, not Store internals.
- Cart prices, totals, Checkout controls, orders, and Stock reduction remain out of scope.
- Dependencies are currently absent. Run `npm ci` before validation.
- The workspace is not a Git repository, so implementation must not include branch or commit steps.

## Planned File Structure

- `src/types.ts`: public Cart, Line item, stored Line item, and Cart error types.
- `src/store.ts`: authoritative Cart lifecycle, validation, mutation, copying, and reset behavior.
- `src/routes/carts.ts`: request validation, Cart endpoints, and domain-error-to-HTTP mapping.
- `src/app.ts`: mount the Cart router before the `/api` fallback.
- `tests/carts.test.ts`: Cart API behavior through `createApp()`.
- `public/index.html`: Cart navigation control, dedicated view, and polite live region.
- `public/app.js`: pure Cart rendering plus browser Cart state, API calls, storage, navigation, and event handling.
- `public/app.d.ts`: declarations for Cart render-helper inputs and exports.
- `public/styles.css`: Cart layout, controls, pending/disabled states, and narrow-screen behavior.
- `tests/render.test.ts`: pure rendering, escaping, availability, quantity limits, empty state, and Cart count tests.

---

## [ ] Phase 1 — First Add Vertical Tracer Bullet

**Spec version:** Approved 2026-11-13 design spec.

**Outcome:** From a Catalogue card, a shopper can add one available Book, receive an opaque Cart identifier, open the Cart view, and see that Book with quantity one.

**Files:**

- Modify: `src/types.ts`
- Modify: `src/store.ts`
- Create: `src/routes/carts.ts`
- Modify: `src/app.ts`
- Create: `tests/carts.test.ts`
- Modify: `public/index.html`
- Modify: `public/app.js`
- Modify: `public/app.d.ts`
- Modify: `public/styles.css`
- Modify: `tests/render.test.ts`

**Interfaces:**

```ts
export interface StoredCartLineItem {
  bookId: string;
  quantity: number;
}

export interface CartLineItem {
  book: Book;
  quantity: number;
}

export interface Cart {
  id: string;
  lineItems: CartLineItem[];
}

export type CartErrorCode =
  | "BOOK_NOT_FOUND"
  | "CART_NOT_FOUND"
  | "LINE_ITEM_NOT_FOUND"
  | "STOCK_EXCEEDED";

export class CartStoreError extends Error {
  constructor(
    public readonly code: CartErrorCode,
    message: string,
  );
}
```

Store methods introduced in this phase:

```ts
createCart(bookId: string, quantity: number): Cart;
getCart(cartId: string): Cart | undefined;
```

Frontend helpers introduced in this phase:

```js
export function cartLineItemHTML(lineItem) {}
export function cartViewHTML(cart) {}
export function cartCount(cart) {}
```

- [ ] Add failing API tests in `tests/carts.test.ts` proving `POST /api/carts` with `{ bookId: "the-lantern-sea", quantity: 1 }` returns `201`, an opaque non-empty `id`, and one Line item containing the current Book and quantity `1`.
- [ ] Add a test proving a created Cart can be read with `GET /api/carts/:cartId`.
- [ ] Add `beforeEach(() => store.reset())` and prove reset makes the previously created Cart return `404`.
- [ ] Run `npm test -- tests/carts.test.ts`; expect failures because Cart routes and Store operations do not exist.
- [ ] Add failing render tests proving an available Book renders an “Add to cart” command, `cartViewHTML` renders the Book title and quantity, and `cartCount` returns the sum of quantities.
- [ ] Run `npm test -- tests/render.test.ts`; expect failures for missing exports or markup.
- [ ] Add the Cart types and `CartStoreError` to `src/types.ts`.
- [ ] Add a private Cart collection to `Store`, clear it in `reset()`, and generate Cart IDs with `randomUUID()` from `node:crypto`.
- [ ] Implement `createCart` as one synchronous operation: resolve the Book, reject unknown Books, reject quantities above current Stock, store only `bookId` and `quantity`, and return a copied Cart containing current Book data.
- [ ] Implement `getCart` so it derives current Book data and returns fresh Cart, Line item, and Book objects.
- [ ] Implement `POST /api/carts` and `GET /api/carts/:cartId` in `src/routes/carts.ts`.
- [ ] Validate that the POST body is a non-array object, `bookId` is a non-empty string, and `quantity` is a JSON number that is a positive integer.
- [ ] Map malformed input to `400`, unknown Books or Carts to `404`, and Stock conflicts to `409`, always using `{ error: string }`.
- [ ] Mount `cartsRouter` in `src/app.ts` before the unknown `/api` handler.
- [ ] Add a Cart navigation button with count, a hidden `#cart-view`, and an `aria-live="polite"` status region to `public/index.html`.
- [ ] Extend Catalogue cards with an available “Add to cart” button carrying the Book ID.
- [ ] Implement the three pure Cart helpers and declare `BookLike`, `CartLineItemLike`, `CartLike`, and their exports in `public/app.d.ts`.
- [ ] Add browser state for the current Cart. On the first add, POST quantity `1`, retain the response model, save its `id` under the fixed key `chapter-one-cart-id`, update the count, render the Cart view, and announce success.
- [ ] Extend view switching so the Cart control hides hero, Catalogue, detail, and About views and shows the dedicated Cart view.
- [ ] Add minimal Cart and navigation-count styling consistent with the existing navy, orange, white, and restrained-border design.

**Validation:**

```text
npm test -- tests/carts.test.ts
npm test -- tests/render.test.ts
npm run typecheck
npm run lint
```

**Acceptance criteria:**

- Creating a Cart returns `201` and exactly one Line item.
- The Cart ID is opaque and generated by the server.
- The created Cart is retrievable through its ID.
- Reset restores Books and removes Carts.
- An available Catalogue Book can be added from the browser.
- The Cart view shows the selected Book and quantity one.
- The navigation count shows one.
- Only `cartId`, not Cart contents, is written to browser storage.

---

## [ ] Phase 2 — Repeat Adds, Multiple Books, and Browser Recovery

**Spec version:** Approved 2026-11-13 design spec.

**Depends on:** Phase 1 Cart types, creation endpoint, Store copying, and first-add browser state.

**Outcome:** A shopper can add to an existing Cart, multiple Books remain together, repeat adds update one Line item, and a stored Cart is restored or discarded correctly after restart.

**Files:**

- Modify: `src/store.ts`
- Modify: `src/routes/carts.ts`
- Modify: `tests/carts.test.ts`
- Modify: `public/app.js`
- Modify: `tests/render.test.ts`

**Interface:**

```ts
addCartItem(cartId: string, bookId: string, quantity: number): Cart;
```

- [ ] Add failing API tests for `POST /api/carts/:cartId/items` proving a different Book creates a second Line item.
- [ ] Add a failing test proving adding the same Book again increases its existing quantity and leaves only one Line item for that `bookId`.
- [ ] Add a failing test proving two Cart IDs cannot read or alter one another’s Line items.
- [ ] Add failing tests proving an unknown Cart or Book returns `404` in the standard error shape without changing an existing Cart.
- [ ] Add a failing test proving an add that would exceed Stock returns `409` and leaves the previous quantity unchanged.
- [ ] Run `npm test -- tests/carts.test.ts`; expect the new cases to fail.
- [ ] Implement `addCartItem` as one synchronous check-and-mutate operation, validating the resulting quantity against current Stock before changing state.
- [ ] Add the POST items route with the same body validation rules as Cart creation and map Store errors consistently.
- [ ] Update add-to-Cart browser behavior to use `POST /api/carts/:cartId/items` when a current Cart ID exists.
- [ ] On bootstrap, read `chapter-one-cart-id`; when present, request `GET /api/carts/:cartId`, replace the browser model, render the view, and update the count.
- [ ] Treat only a bootstrap `404` as stale Cart recovery: remove the stored ID, clear the model, render an empty Cart, and allow the next add to create a new Cart.
- [ ] Keep non-404 bootstrap failures visible in the live region and retain the stored ID.
- [ ] Add a render test proving two Line items render together and count uses total quantities rather than distinct Line items.
- [ ] Ensure successful repeat adds replace the entire browser Cart model with the server response and announce success.

**Validation:**

```text
npm test -- tests/carts.test.ts
npm test -- tests/render.test.ts
npm run typecheck
npm run lint
```

**Acceptance criteria:**

- Different Books create separate Line items in one Cart.
- Repeat adds increase one existing Line item without duplication.
- A Stock-exceeding repeat add returns `409` without mutation.
- Carts are isolated by opaque ID.
- Reloading restores a server-owned Cart from its stored ID.
- A stale ID is cleared only after `GET` returns `404`.
- Other recovery errors remain visible and do not silently discard the Cart.

---

## [ ] Phase 3 — Absolute Quantity Updates and Removal

**Spec version:** Approved 2026-11-13 design spec.

**Depends on:** Phase 2 multi-item Cart and browser model replacement.

**Outcome:** A shopper can set an absolute whole-number quantity within current Stock or remove a Line item, including the final one.

**Files:**

- Modify: `src/store.ts`
- Modify: `src/routes/carts.ts`
- Modify: `tests/carts.test.ts`
- Modify: `public/app.js`
- Modify: `public/app.d.ts`
- Modify: `public/styles.css`
- Modify: `tests/render.test.ts`

**Interfaces:**

```ts
updateCartItem(cartId: string, bookId: string, quantity: number): Cart;
removeCartItem(cartId: string, bookId: string): void;
```

- [ ] Add failing API tests proving PATCH replaces, rather than increments, the Line item quantity.
- [ ] Add failing tests for unknown Cart, unknown Book, and Book absent from the Cart, expecting `404`.
- [ ] Add a failing test proving an over-Stock PATCH returns `409` and leaves the Cart unchanged.
- [ ] Add failing DELETE tests proving success returns `204` with no body and the Cart remains retrievable without the removed item.
- [ ] Add a failing test proving removing the last Line item leaves an existing empty Cart.
- [ ] Run `npm test -- tests/carts.test.ts`; expect the PATCH and DELETE cases to fail.
- [ ] Implement `updateCartItem` with Book, Cart, Line item, positive-integer, and current-Stock checks completed before mutation.
- [ ] Implement `removeCartItem` so it removes only the requested Line item and never deletes the Cart.
- [ ] Add `PATCH /api/carts/:cartId/items/:bookId` and `DELETE /api/carts/:cartId/items/:bookId`.
- [ ] Return the updated Cart from PATCH and no body from successful DELETE.
- [ ] Add failing render tests for a stable accessible quantity label, `min="1"`, `max` equal to current Book Stock, Update quantity, Remove, and the empty Cart state.
- [ ] Run `npm test -- tests/render.test.ts`; expect failures for missing controls.
- [ ] Extend `cartLineItemHTML` with the number input and commands, escaping IDs and Book fields.
- [ ] Submit the input’s absolute numeric value through PATCH and replace the browser Cart model with the response.
- [ ] On successful DELETE, issue `GET /api/carts/:cartId` and render that authoritative response.
- [ ] Ensure removing the final Line item renders the empty state while retaining the same Cart ID.
- [ ] Disable only the submitted row command while PATCH or DELETE is pending; restore it after success or failure.
- [ ] Announce successful updates, removals, and errors through the polite live region.

**Validation:**

```text
npm test -- tests/carts.test.ts
npm test -- tests/render.test.ts
npm run typecheck
npm run lint
```

**Acceptance criteria:**

- PATCH sets an absolute quantity from one through current Stock.
- Zero is rejected rather than interpreted as removal.
- Failed updates do not alter Cart state.
- DELETE returns `204` and removes only the requested Line item.
- An empty Cart remains retrievable.
- Quantity controls expose current server-derived Stock limits.
- Only the affected command is disabled during each request.

---

## [ ] Phase 4 — Validation, Unavailable Books, and Safe Error Rendering

**Spec version:** Approved 2026-11-13 design spec.

**Depends on:** All Cart endpoints and frontend mutations from Phases 1–3.

**Outcome:** Every endpoint follows the complete validation/status contract, unavailable Books cannot be offered for addition, and unsafe text cannot become markup.

**Files:**

- Modify: `src/routes/carts.ts`
- Modify: `tests/carts.test.ts`
- Modify: `public/app.js`
- Modify: `public/app.d.ts`
- Modify: `public/styles.css`
- Modify: `tests/render.test.ts`

- [ ] Add table-driven API tests for missing bodies, arrays, null, empty `bookId`, missing fields, string quantities, fractional quantities, zero, negative values, `NaN`-inexpressible alternatives such as `null`, and malformed JSON where supported by Express.
- [ ] Assert every invalid body returns `400` with exactly an object containing an `error` string and does not mutate an existing Cart.
- [ ] Add tests proving zero-Stock creation, repeat add, and update return `409` without mutation.
- [ ] Add tests confirming extra request fields do not affect valid behavior.
- [ ] Run `npm test -- tests/carts.test.ts`; expect any uncovered validation cases to fail.
- [ ] Consolidate route validation in a small local parser returning validated `{ bookId, quantity }` or `{ quantity }` values without moving HTTP validation into the Store.
- [ ] Add a single Cart error response mapper covering `404` resource errors and `409` Stock conflicts without exposing stack traces.
- [ ] Add failing render tests proving zero-Stock Books render a disabled “Unavailable” command in both Catalogue and detail views.
- [ ] Add render tests with malicious Book title, author, cover path, ID, and API error text, asserting raw markup is absent.
- [ ] Run `npm test -- tests/render.test.ts`; expect unavailable or escaping cases to fail.
- [ ] Add “Add to cart” to the detail view and use the same available/unavailable behavior as Catalogue cards.
- [ ] Render API error text with `textContent`; if a helper includes error HTML, pass it through `escapeHtml`.
- [ ] For `409`, retain the visible Cart and announce that the requested quantity exceeds current Stock.
- [ ] Ensure failed mutations do not replace or clear the current browser Cart model.
- [ ] Style disabled unavailable and pending controls distinctly while preserving readable contrast.

**Validation:**

```text
npm test -- tests/carts.test.ts
npm test -- tests/render.test.ts
npm run typecheck
npm run lint
```

**Acceptance criteria:**

- All malformed, fractional, zero, and negative quantities return `400`.
- Unknown Cart, Book, or Line item returns `404`.
- Stock conflicts return `409`.
- Every failure uses `{ error: string }` and leaves Cart state unchanged.
- Zero-Stock Books have disabled “Unavailable” commands in both entry points.
- Book and API-provided text cannot execute or render as injected markup.
- A `409` leaves the Cart visible and usable.

---

## [ ] Phase 5 — Responsive Completion and Full Verification

**Spec version:** Approved 2026-11-13 design spec.

**Depends on:** Complete API and storefront behavior from Phases 1–4.

**Outcome:** The Cart experience is responsive, accessible, regression-tested, formatted, and ready for gate 2 approval and subsequent implementation review.

**Files:**

- Modify: `public/styles.css`
- Modify: `public/app.js`
- Modify: `tests/render.test.ts`
- Modify: `tests/carts.test.ts`
- Review without unrelated edits: all files changed in Phases 1–4
- Future review artifact: `docs/superpowers/reviews/f1-build-manage-cart-review.md`

- [ ] Add any missing regression assertions for existing Book Catalogue and detail rendering.
- [ ] Verify Cart rows show cover, title, author, and quantity but no price, Line item total, full total, or Checkout control.
- [ ] Add CSS so Cart rows use stable desktop columns and stack Book information and controls below `720px` without horizontal scrolling.
- [ ] Ensure buttons, inputs, labels, live-region behavior, focus order, and disabled states remain usable with keyboard navigation.
- [ ] Verify every top-level navigation path hides all unrelated views and returning to Home restores the existing Catalogue behavior.
- [ ] Review all fetch paths to ensure non-OK responses are handled and pending controls are restored in `finally`.
- [ ] Review all successful mutations to ensure rendering comes from the returned or freshly fetched server Cart.
- [ ] Run the focused Cart API and render suites.
- [ ] Run the complete repository validation suite after `npm ci`.
- [ ] Start the application with `npm run dev` and manually verify first add, repeat add, multiple Books, update, removal, empty state, reload restoration, stale-ID recovery after restart, unavailable controls, and narrow-screen layout.
- [ ] Record final review findings at the supplied review destination during the later review workflow; do not create that artifact during planning.

**Validation:**

```text
npm test -- tests/carts.test.ts
npm test -- tests/render.test.ts
npm test
npm run typecheck
npm run lint
npm run format:check
npm audit --audit-level=high
```

**Acceptance criteria:**

- All approved F1 done conditions are observable through the API and storefront.
- Cart rows stack without horizontal scrolling on narrow screens.
- Existing Catalogue, detail, About, health, and Book API behavior remains green.
- No Cart prices, totals, Checkout, Stock reduction, accounts, sessions, or persistence have been introduced.
- All five project checks and the high-severity audit complete successfully.
- Any environment-related inability to run a check is recorded explicitly rather than reported as passing.

## Open Questions

None block F1. The approved design resolves Cart behavior. Shopper details, collection deadlines, Stock release policy, and outcome measurement remain deferred to Checkout, Order handling, or release measurement.

Execution prerequisites remain: install the existing lockfile dependencies with `npm ci`. Branch, commit, and remote operations are unavailable because the workspace is not a Git repository.
