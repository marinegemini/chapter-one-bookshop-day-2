# Intent — Online purchasing

Status: Approved
Approved by: Workspace user (via chat)
Date: 2026-11-13
Sources: `docs/inputs/shopper-feedback.md` (FB-01–FB-10, ON-01–ON-05)

## Problem

Shoppers can browse the catalogue but cannot complete a purchase on the site.
They must instead describe wanted books and quantities by email, which makes
multi-book requests, changes, pricing, availability, and confirmation unclear.
The owner spends about an hour each day managing these threads and still makes
stock mistakes, including promising books that are unavailable.

## Pain points

| ID | Pain point | Evidence |
| --- | --- | --- |
| P1 | Shoppers cannot buy books through the site, and email ordering consumes owner time. | FB-01, ON-01 |
| P2 | Selecting multiple books and correcting quantities by email is unreliable and prone to accidental duplicates. | FB-02, FB-04, FB-10 |
| P3 | Shoppers can request unavailable books, and manual handling can promise more copies than are in stock. | FB-03, ON-01, ON-03 |
| P4 | Shoppers cannot review their books, quantities, and full total before committing. | FB-05 |
| P5 | Shoppers receive no immediate confirmation that an order succeeded or what it contains. | FB-07 |
| P6 | Low stock is not prominent enough to help shoppers judge availability. | FB-06 |
| P7 | Shoppers who remember an author but not a title cannot easily find the book. | FB-08 |
| P8 | Some shoppers want gift wrapping and a gift note. | FB-09, ON-05 |

## Desired outcomes

- Shoppers can place a multi-book order without an email exchange or an account.
- Shoppers can correct their selection and see the full total before checkout.
- Successful checkout never reduces stock below zero.
- Shoppers immediately know whether checkout succeeded and what they ordered.
- The owner spends less time translating email requests into sales and correcting stock mistakes.

## Constraints

- Payment is on collection; online card payment is not required. (ON-02)
- Shoppers must not need a customer account. (ON-04)
- Stock must never go below zero. (ON-03)
- Gift wrapping must not delay the ability to buy books. (ON-05)
- Cart and order state may be temporary and in memory; no users, authentication,
  sessions, database, or payment provider are introduced for this scope.

## Features

Priorities use **Must**, **Should**, and **Later**. Must features form the first
usable online purchasing flow; Should features improve discovery and urgency;
Later features are explicitly deferred.

| ID | Feature | Solves | Priority |
| --- | --- | --- | --- |
| F1 | Build and manage a cart | P1, P2 | Must |
| F2 | Review the cart and total | P4 | Must |
| F3 | Stock-safe checkout for collection | P1, P3 | Must |
| F4 | Order confirmation | P5 | Must |
| F5 | Low-stock visibility | P6 | Should |
| F6 | Find books by author | P7 | Should |
| F7 | Gift options | P8 | Later |

### F1 — Build and manage a cart

- **User value:** A shopper can collect multiple books and set the quantity of
  each before checkout, avoiding an email thread and accidental duplicate lines.
- **Done when:**
  - A shopper can add an available book from the catalogue to a cart.
  - A shopper can view all selected books and their quantities together.
  - Adding the same book again updates one line item rather than creating a
    duplicate line item.
  - A shopper can change a line item's quantity or remove it before checkout.
  - A quantity cannot be set below one or above the currently available stock.
- **Not in this feature:** Checkout, payment, accounts, or preserving a cart
  across server restarts.

### F2 — Review the cart and total

- **User value:** A shopper can verify exactly what they are about to order and
  what they will pay before committing.
- **Done when:**
  - The cart shows each book, its quantity, its unit price, and its line total.
  - The cart shows a full total for all line items.
  - Totals update when a shopper changes a quantity or removes a line item.
  - Checkout cannot proceed from an empty cart.
- **Not in this feature:** Delivery charges, discounts, taxes added at checkout,
  or online payment.

### F3 — Stock-safe checkout for collection

- **User value:** A shopper can commit the cart as an order for collection and
  know unavailable stock will not be promised.
- **Done when:**
  - A shopper can check out a non-empty cart without creating an account or
    making an online card payment.
  - Checkout succeeds only when every requested quantity is available at the
    moment of checkout.
  - A successful checkout creates one order containing the cart's books,
    quantities, and total.
  - A successful checkout reduces stock by the ordered quantities.
  - Concurrent or repeated checkout attempts cannot reduce stock below zero.
  - If any quantity is unavailable, checkout does not create an order or reduce
    stock and tells the shopper what must be corrected.
- **Not in this feature:** Card payment, delivery, customer accounts, or
  long-term order storage.

### F4 — Order confirmation

- **User value:** A shopper immediately knows the order succeeded and can verify
  what was ordered.
- **Done when:**
  - After successful checkout, the shopper sees an order reference, each ordered
    book and quantity, the full total, and that payment is due on collection.
  - A failed checkout is not presented as confirmed and provides a clear reason.
- **Not in this feature:** Email or SMS notifications, payment receipts, or
  order-history accounts.

### F5 — Low-stock visibility

- **User value:** A shopper can recognize books likely to sell out and make an
  informed choice sooner.
- **Done when:**
  - Catalogue books with three or fewer copies available have a prominent,
    understandable low-stock indication.
  - Books with no available copies are clearly identified as unavailable and
    cannot be added to a cart.
- **Not in this feature:** Restock alerts, reservations, or stock forecasting.

### F6 — Find books by author

- **User value:** A shopper who remembers an author can find matching books
  without knowing the title.
- **Done when:**
  - A shopper can enter all or part of an author's name and see matching
    catalogue books.
  - Matching does not depend on letter case.
  - A no-results state clearly tells the shopper that no books matched.
- **Not in this feature:** Full-text search, recommendations, or advanced
  filtering.

### F7 — Gift options

- **User value:** A shopper can request gift wrapping and include a gift note.
- **Done when:** To be defined when this later feature is prioritized.
- **Not in this feature:** This feature is not part of the initial online
  purchasing release.

## Out of scope

- Online card payments or a payment provider.
- Customer accounts, authentication, and order history.
- Delivery and shipping.
- Persistent carts or orders across server restarts.
- Gift wrapping and gift notes in the initial release.

## Assumptions

- “Everything” in the requested total means book line totals only because the
  source specifies collection and does not mention delivery fees, taxes, or
  discounts.
- The first release is ordered Must features first, followed by Should features;
  the source does not assign a more detailed sequence within those groups.

## Open questions

- What shopper details, if any, are required so the owner can identify an order
  at collection?
- Should an order reserve stock indefinitely, or is a collection deadline and
  stock-release policy needed?
- What evidence will be used to measure the reduction in owner time and stock
  mistakes after release?
