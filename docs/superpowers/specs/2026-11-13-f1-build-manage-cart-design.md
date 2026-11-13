# F1 — Build and manage a cart: design spec

Status: Approved
Design approval: Approved by Workspace user via chat on 2026-11-13
Requirements approval: Approved by Workspace user via chat on 2026-11-13
Source: `docs/intent.md`, F1 — Build and manage a cart
Intent sources: `docs/inputs/shopper-feedback.md` (FB-01–FB-10, ON-01–ON-05)
Traces to: F1

## Purpose

Give an anonymous shopper a Cart in which they can collect available Books, see all selected Line items together, correct quantities, and remove Line items before checkout. The design keeps stock validation on the server and fits the existing Express, vanilla JavaScript, and in-memory Store architecture.

## Scope

This feature includes:

- adding an available Book from the Catalogue or Book detail view;
- creating an anonymous Cart on the first successful add;
- viewing the Cart's Line items and quantities in one dedicated view;
- adding the same Book again by increasing its existing Line item's quantity;
- setting a Line item's quantity to a whole number within current Stock;
- removing a Line item;
- clear success, empty, loading, and error states; and
- retaining the opaque `cartId` in the same browser across page reloads while the server-side Cart exists.

This feature excludes Checkout, payment, accounts, authentication, sessions, prices and totals in the Cart, orders, stock reservation or reduction, and preserving Cart contents across server restarts. F2 owns Cart prices and totals. F3 owns final Stock checks and Stock reduction at Checkout.

## Approaches considered

### 1. Server-managed anonymous carts identified by `cartId` — recommended

The Store owns Carts and returns an opaque `cartId`. The browser retains that identifier and sends it on later requests. Routes validate input and delegate Cart operations to the Store.

This keeps the server authoritative for Stock limits, isolates anonymous shoppers from one another, and creates the right boundary for later Checkout without introducing accounts or sessions. The trade-off is handling an unknown or stale `cartId`, especially after a server restart.

### 2. One process-wide Cart

A singleton Cart would be the smallest backend change, but every browser would share it. One shopper could see or alter another shopper's selection, so this does not meet the implied isolation of “a shopper's selected books.”

### 3. Browser-only Cart

Keeping Line items entirely in browser storage would avoid Cart routes, but Stock validation would be duplicated or trusted to the client and later Checkout would need a second Cart model. It also weakens HTTP-level behavior tests and the Store's role as runtime source of truth.

## Architecture

### Domain types

Add a stored Cart representation with:

- `id`: an opaque, server-generated identifier;
- `lineItems`: an array of stored Line items, each containing `bookId` and integer `quantity`.

A Cart response contains `id` and `lineItems`. Each response Line item contains the current Book representation plus `quantity`, allowing the frontend to show Book identity and enforce the current Stock maximum. The Store records only `bookId` and `quantity`; it derives current Book data when returning a Cart so Catalogue data is not duplicated.

A Cart has at most one Line item per `bookId`. An empty Cart is valid after its last Line item is removed.

### Store responsibilities

Extend the existing Store as the single source of truth for Books and Carts. It will:

- reset both Books and Carts in `reset()`;
- create a Cart with its first Line item;
- retrieve a Cart by identifier;
- add a Book to a Cart, increasing the existing Line item's quantity when present;
- set a Line item's absolute quantity;
- remove a Line item; and
- check the Book exists and the resulting quantity is between 1 and current Stock before each add or update mutation.

Validation and mutation happen in one synchronous Store operation so a failed check cannot partially change a Cart. Cart operations do not reserve or reduce Stock. Returned values are copies so callers cannot mutate Store state by reference.

Opaque identifiers can use the Node runtime's built-in UUID support; no dependency is required.

### HTTP API

Add a Cart router mounted through `createApp()`:

| Method and path | Request | Success | Behavior |
| --- | --- | --- | --- |
| `POST /api/carts` | `{ "bookId": string, "quantity": 1 }` | `201` with the created Cart | Creates a Cart and its first Line item after validating the Book and quantity. |
| `GET /api/carts/:cartId` | None | `200` with the Cart | Returns all selected Books and quantities together. |
| `POST /api/carts/:cartId/items` | `{ "bookId": string, "quantity": 1 }` | `200` with the updated Cart | Adds the quantity to a new or existing Line item; it never creates a duplicate Line item. |
| `PATCH /api/carts/:cartId/items/:bookId` | `{ "quantity": integer }` | `200` with the updated Cart | Replaces the Line item's quantity with the supplied absolute value. |
| `DELETE /api/carts/:cartId/items/:bookId` | None | `204` with no body | Removes the Line item. |

Request-body validation occurs in the router before the Store is called. `bookId` must be a non-empty string and `quantity` must be a JSON number that is a positive integer. Extra fields do not affect behavior.

All failures use `{ "error": "message" }`:

- `400` for a missing or malformed body, a non-integer quantity, or a quantity below one;
- `404` for an unknown Cart, unknown Book, or Book not present in the Cart being changed or removed; and
- `409` when an add or update would make the quantity exceed current Stock, including adding a Book with zero Stock.

A failed request leaves the Cart unchanged. Error messages are shopper-readable but tests rely primarily on status and error shape rather than exact prose.

## Frontend experience

### Entry points and navigation

Each Catalogue card gains an “Add to cart” command alongside “View Book.” The Book detail view also gains “Add to cart.” A Book with zero Stock has a disabled “Unavailable” command so the browser does not offer an invalid add.

The top navigation gains a Cart control with a count equal to the sum of Line item quantities. Activating it opens a dedicated Cart view and hides the hero, Catalogue, Book detail, and About views. Returning to the Catalogue uses the existing view-switching model.

The browser keeps only the opaque `cartId` in `localStorage`; Cart contents remain server-owned. On bootstrap, a stored identifier triggers `GET /api/carts/:cartId`. If the identifier is unknown, as after a server restart, the browser clears it and shows an empty Cart state. The next successful add creates a new Cart. Other request failures remain visible and do not silently discard the Cart.

### Cart view

The Cart view lists all Line items together. Each row shows the Book cover, title, author, and current quantity. It provides:

- a number input with minimum `1`, maximum equal to current Stock, and a stable accessible label;
- an “Update quantity” command; and
- a “Remove” command.

Submitting a quantity sends its absolute value through the `PATCH` endpoint. Removing the last Line item leaves the Cart in the empty state rather than deleting the Cart. Cart prices, Line item totals, full totals, and Checkout controls are absent until their owning features.

On narrow screens, each Line item stacks its Book information and controls without horizontal scrolling. The design extends the existing navy, orange, white, and restrained-border visual language in `public/styles.css`; it does not introduce another stylesheet or nested decorative cards.

### Interaction feedback

Only the affected command is disabled while its request is pending, preventing accidental duplicate submissions. A polite live region announces successful adds, quantity changes, removals, and request errors. After a successful mutation, the response replaces the browser's Cart model and both the Cart view and navigation count are rendered from that model.

For a `409`, the Cart remains visible and the message tells the shopper that the requested quantity exceeds current Stock. The maximum shown by the input updates from the latest Cart response when available; the server remains authoritative if Stock changes between display and submission.

### Rendering and safety

New render helpers remain pure and exported, including helpers for the Cart view and its Line items. All Book fields and API error text interpolated into HTML pass through `escapeHtml`. Event wiring, fetch calls, storage access, and DOM mutation stay outside render helpers. Bootstrap remains guarded so importing `public/app.js` in Vitest has no side effects.

## Data flow

### First add

1. The shopper activates “Add to cart” with no current `cartId`.
2. The frontend sends `POST /api/carts` with the Book identifier and quantity `1`.
3. The route validates the body and asks the Store to create the Cart.
4. The Store checks the Book and current Stock, creates one Line item, and returns the Cart.
5. The frontend stores the returned `cartId`, replaces its Cart model, updates the count, and announces success.

### Repeat add

1. The frontend sends `POST /api/carts/:cartId/items` with the Book identifier and quantity `1`.
2. The Store calculates the existing quantity plus one and validates the result against current Stock.
3. On success, the existing Line item is updated in place and the returned Cart has no duplicate.
4. On conflict, the Cart is unchanged and the frontend shows the API error.

### Quantity change or removal

A quantity change sends the desired absolute quantity through `PATCH`; zero is not shorthand for removal. Removal always uses `DELETE`, making the below-one rule unambiguous. Each successful operation rerenders from the server result, or locally removes the item after a successful `204` and then refreshes the Cart with `GET` to preserve the server as source of truth.

## Testing strategy

API behavior tests use Supertest against `createApp()` and call `store.reset()` before each test. They cover:

- first add creates a Cart containing one Line item;
- an available Book can be added;
- adding the same Book again increases one Line item's quantity without duplication;
- a different Book creates a second Line item;
- retrieving a Cart returns all selected Books and quantities;
- setting a valid absolute quantity succeeds;
- removing a Line item succeeds and an empty Cart remains retrievable;
- malformed, fractional, zero, and negative quantities return `400` without mutation;
- unknown Cart, Book, and Line item cases return `404` in the standard JSON shape;
- unavailable or over-Stock adds and updates return `409` without mutation; and
- `store.reset()` removes created Carts as well as restoring the Catalogue.

Frontend render tests import pure helpers and verify:

- available and unavailable add commands render correctly;
- multiple Line items and their quantities render together;
- quantity controls expose the correct minimum and current Stock maximum;
- the empty Cart state renders;
- Book and API-provided text is escaped; and
- the Cart count is derived from total quantities, not distinct Line items.

No test starts a real HTTP listener or asserts private Store implementation details.

## Delivery shape

Implementation should proceed as a vertical tracer bullet: define the minimal Cart types and Store operation, expose first-add creation through the route, prove it with an HTTP test, and render the resulting Cart path in the storefront. Subsequent increments add repeat-add behavior, quantity updates, removal, error states, and full render coverage. This sequencing is guidance for the later plan, not an implementation plan in this artifact.

## Done when

- From the Catalogue, a shopper can add a Book whose Stock is at least one and then see it in their Cart.
- The Cart view shows every selected Book and its quantity together.
- Adding the same Book again increases its existing Line item's quantity and does not create a duplicate Line item.
- A shopper can set a Line item's quantity to any whole number from one through that Book's current Stock.
- A shopper can remove a Line item, including the final Line item, and then sees the empty Cart state.
- The API rejects quantities below one, fractional quantities, and quantities above current Stock without changing the Cart.
- A Book with zero Stock cannot be added.
- Cart operations for missing resources and invalid requests return the specified HTTP status with `{ "error": "message" }`.
- Cart state is isolated by opaque `cartId`, resets with the in-memory Store, and does not introduce accounts, sessions, a database, or another persistence layer.
- The storefront handles a stale `cartId` after server restart by clearing it and allowing a new Cart on the next add.
- Automated API and pure-render tests cover the observable behaviors above.

## Unresolved product decisions

None block F1. The approved requirements determine the Cart behaviors in this spec. Shopper details, collection deadlines, Stock release policy, and outcome measurement remain open in `docs/intent.md`, but they belong to Checkout, Order handling, or release measurement rather than Cart management.

## Approval

This draft is ready for human gate 1 review. Approval authorizes creation of the separate implementation plan at `docs/superpowers/plans/2026-11-13-f1-build-manage-cart.md`; it does not authorize implementation by itself.
