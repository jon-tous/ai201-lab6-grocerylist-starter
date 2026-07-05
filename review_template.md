# Code Review Notes

Fill this in as you work through the milestones. Each section mirrors the structure of a real GitHub pull request review.

---

## PR #1 — Bulk Purchase (`pr1_bulk_purchase.py`)

### Summary
This PR adds a bulk purchase endpoint that is intended to mark every item in a list as purchased in one request. The implementation is simple, but it does not correctly handle the intended semantics for only purchasing unpurchased items and attributing the purchase to the requesting user.

### Issues

For each issue you find, note: where it is (file + function), what's wrong, and why it matters in production.

**Issue 1**
- Location: [prs/pr1_bulk_purchase.py](prs/pr1_bulk_purchase.py), `purchase_all_items()`
- What's wrong: The function queries `Item.query.filter_by(list_id=list_id).all()` and processes every item in the list, including items that are already purchased.
- Why it matters: Already-purchased items get overwritten as if they were newly purchased, which corrupts purchase history and makes the endpoint behave incorrectly in production.
- Suggested fix: Filter the query to only unpurchased items, such as `Item.query.filter_by(list_id=list_id, is_purchased=False).all()`.

**Issue 2**
- Location: [prs/pr1_bulk_purchase.py](prs/pr1_bulk_purchase.py), `purchase_all_items()`
- What's wrong: The loop sets `item.purchased_by = user_id` for every processed item, even if the item was previously purchased by a different user. In the live test, the previously Leo-purchased Olive Oil item was overwritten to Maya.
- Why it matters: This breaks the integrity of purchase attribution and can incorrectly assign purchases to the wrong person.
- Suggested fix: Only update `purchased_by` for items that are actually being newly purchased, and preserve existing purchase metadata for already-purchased items.

**Issue 3**
- Location: [prs/pr1_bulk_purchase.py](prs/pr1_bulk_purchase.py), `purchase_all()`
- What's wrong: The route reads `user_id = data.get("user_id")` but does not validate that the field is present. A request with `{}` still succeeds and sets `purchased_by` to `None`.
- Why it matters: The endpoint accepts invalid input and silently creates purchases with no responsible user, which is unsafe and confusing in production.
- Suggested fix: Return a 400 error if `user_id` is missing or empty, and require a valid user ID before performing the bulk purchase.

**Issue 4**
- Location: [prs/pr1_bulk_purchase.py](prs/pr1_bulk_purchase.py), `purchase_all_items()`
- What's wrong: The function returns `len(items)`, where `items` is the full list of items returned by the query rather than the count of newly purchased items.
- Why it matters: The response reported `8` even though the endpoint should only count the newly purchased items that changed state. This makes the endpoint’s API misleading.
- Suggested fix: Count only the items that were actually updated, for example with a counter incremented inside the loop for unpurchased items.

### Questions for the Author
- Should the endpoint treat already-purchased items as a no-op, or should it explicitly error if the list already contains purchased items?
- Do you want the endpoint to require a valid `user_id` and return a client error when it is missing?
- Is the intended response supposed to represent “items changed” or “items present in the list”?

### Verdict
- [ ] Approve — ship it
- [x] Request Changes — needs fixes before merging
- [ ] Comment — needs discussion before a verdict

**Rationale** *(1–2 sentences)*:
The live behavior showed that the endpoint overwrites existing purchase history, accepts missing user IDs, and returns the wrong count. These are material correctness issues, so I would not approve it until they are fixed.

---

## PR #2 — List Stats (`pr2_list_stats.py`)

### Summary
This PR adds a stats endpoint for the active shopping view so the frontend can show how many items remain and how those remaining items are distributed by category. The implementation is close in spirit, but it does not match the frontend request in a couple of important ways.

### Issues

**Issue 1**
- Location: [prs/pr2_list_stats.py](prs/pr2_list_stats.py), `get_list_stats()`
- What's wrong: The `by_category` breakdown is computed from all items in the list, not just the unpurchased items that are still remaining. In the live test, the endpoint returned `by_category` values summing to `8`, while `remaining` was `0`.
- Why it matters: This breaks the frontend use case. A shopper viewing the active list needs to see what is still left to buy by section, not a count of everything that has ever been on the list.
- Suggested fix: Build the category breakdown from the remaining/unpurchased items only, so that `sum(by_category.values())` matches `remaining` and the frontend can display accurate shopping guidance.

**Issue 2**
- Location: [prs/pr2_list_stats.py](prs/pr2_list_stats.py), `list_stats()` / `get_list_stats()`
- What's wrong: The endpoint returns `200 OK` with zeroed-out stats for a non-existent list ID instead of failing like the existing items endpoint does. The live test showed `{"error":"List 'bad-list-id' not found"}` for `GET /lists/<list_id>/items`, but the stats route returned `200` with empty stats.
- Why it matters: Inconsistent error handling makes the API harder to reason about and can cause downstream clients to silently treat missing data as a valid empty list.
- Suggested fix: Check whether the list exists before computing stats and return a `404` response (with an error payload) when it does not.

### Questions for the Author
- Should the category breakdown represent only unpurchased items, or is the intention to show the full history of the list by category?
- Do you want the stats endpoint to match the existing list-items route’s error semantics and return `404` for unknown list IDs?

### Verdict
- [ ] Approve — ship it
- [x] Request Changes — needs fixes before merging
- [ ] Comment — needs discussion before a verdict

**Rationale** *(1–2 sentences)*:
The category breakdown is semantically wrong for the frontend’s “what remains to buy” use case, and the missing-list behavior is inconsistent with the rest of the API. I would request changes before approving.

---

## Reflection

*Answer after completing both reviews.*

**1.** Which issue was hardest to spot, and why?

>

**2.** Which issues do you think an LLM reviewer (like Claude reviewing its own code) would most likely miss? Why?

>

**3.** One thing you'd add to a code review checklist for AI-generated backend code:

>
