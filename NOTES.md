## NOTES.md

### Backend

#### BE-1 — Checkout error (500)

**What I checked:** Traced `POST /v1/th/orders` starting from the router (`app/api/orders.py`) into `OrderService.create_from_cart()`, then read through every related piece (`CartService`, `StockService`, `InMemoryDB`, Pydantic schemas) to rule out `TypeError`/`AttributeError`/`ValidationError` one by one.

**What was wrong:** After building `first_line = lines[0].model_dump()`, the code looked up `meal = self.db.meals[first_line["mealId"]]` and then discarded the result entirely (`meal.store_id` was assigned to `_`). This block had no effect on the response, the order object, or any log — it was dead code left over from an unfinished "receipt / downstream notifications" idea. Worse, the lookup key didn't always match an entry in `self.db.meals`, causing an uncaught `KeyError` → FastAPI returned a 500.

**Fix:** Removed the dead code block entirely, rather than just fixing the key. The rest of the checkout flow (building the `OrderLine`, creating the `Order`, saving to the db, then `self.cart.clear(user_id, release_stock=False)`) was already correct and needed no further changes.

**Verified:** Order placed successfully (order number `SF-1001` appears in stock events with no 500), cart was empty afterward, and no `INCREMENT` event fired during checkout — stock only moves on add-to-cart (`DECREMENT`) and on cancel (`INCREMENT`, see BE-3).

#### BE-2 — Wrong price bug

**What I checked:** How per-line prices in the cart are calculated (`unit_price`, `line_total`, `subtotal`).

**What was wrong:** Pricing used `meal.original_price` instead of `meal.discounted_price`, so customers were charged full price.

**Fix:** Changed to `unit_price = meal.discounted_price`, `line_total = unit_price * quantity`, with `subtotal` summed from `line_total`. Verified `meal_1 × 2 → subtotal = 158`.

#### BE-3 — Inventory bug on cancel

**What I checked:** Followed the repro steps (reset → add `meal_2` qty 3 → place order → cancel) and watched stock at each step.

**What was wrong:** Cancel changed the status to `CANCELLED` but never restored stock correctly.

**Fix:** On cancel, loop through `order.lines` and call `stock.apply(..., event_type=INCREMENT, ...)` for each line's quantity to restore stock before setting `order.status = CANCELLED`. Verified: after placing the order, `meal_2` stock dropped to `2`, and after cancel it returned to `5`.

#### BE-4 — Inventory bug when increasing cart quantity

**What I checked:** Followed the repro steps (reset → add `meal_1` qty 1 → update to qty 3) and watched stock events after each step.

**What was wrong:** Updating the quantity of an item already in the cart didn't reserve/release stock based on the *difference* in quantity — the increment/decrement amount was calculated incorrectly.

**Fix:** Calculate `delta = new_quantity - old_quantity`. If `delta > 0`, decrement stock by `delta` (reserve more). If `delta < 0`, increment stock by `abs(delta)` (release the difference). Verified: qty 1 → stock `9`; qty 1 → 3 → stock `7`.

### Frontend

#### FE-1 — Wrong "You pay" price

**What I checked:** The price display section in the meal card component.

**What was wrong:** The "you pay" amount used `meal.original_price` instead of `meal.discounted_price`.

**Fix:** Kept `original_price` shown with a strikethrough as before, and changed the bold "pay" figure to `meal.discounted_price`.

#### FE-2 — Stock events filter

**What I checked:** Traced the whole chain end to end — `StockEventsPage` component → `api.listStockEvents(meal_id)` client → `GET /stock-events` router (`app/api/stock.py`) → `StockService.list_events()`.

**What was wrong:** The API client (`api.listStockEvents`) sent the query parameter under the wrong name — `?meal=<id>` instead of `?meal_id=<id>`, which is what the backend router expected. Every other layer was already correct (the input is a controlled field, the "Apply filter" button passes the current `mealFilter` into `load()`, the router forwards `meal_id` straight into `list_events(meal_id=...)`, and the service filters with `[e for e in events if e.meal_id == meal_id]` correctly). But because the query param name didn't match, the backend never actually received the filter value (`meal_id` was always `None`), so the filter didn't work as intended.

**Fix:** Changed the query string in the API client from `?meal=` to `?meal_id=${encodeURIComponent(meal_id)}` to match the parameter name used by the router and `StockService.list_events()`.

**Verified:** Reset data, added `meal_1` and `meal_2` to the cart to generate events for both meals, then filtered by `meal_1` on the Stock events page — the table correctly filtered from 6 mixed events down to 3 events, all with `meal_id: meal_1`.