# Problem Set 1 —  Reading and Writing Concepts

## Exercise 1: Reading a Concept — GiftRegistration

### Q1. Invariants
- **Invariant 1:** For each request, the remaining amount plus the sum of purchased amounts equals the total requested so far for that request.
- **Invariant 2:** Every Purchase belongs to exactly one Request in exactly one Registry, and for that request `purchase.Item == request.Item`.

The more important invariant is Invariant 1, because it prevents overselling and directly realizes the core requirement that purchases never exceed what has been requested. The action whose design is most constrained by this is `purchase`, as its `requires` clause demands that the matching request has at least the requested `count` remaining, and its `effects` clause both decrements `request.count` and adds a purchase of that count, preserving the accounting equality.

### Q2. Fixing an action
The `removeItem` action can violate the invariants: if a request is removed after purchases have been recorded for that item, those purchases would no longer be connected to any request, so tracking breaks. The fix is to tighten the precondition so a request can be removed only if it has no purchases. More generally, any action that modifies a request after purchases exist must preserve the aggregation invariant.
### Q3. Inferring behavior
Yes, a registry can be opened and closed repeatedly. The `open` action requires that the registry exists and is not active, and the `close` action requires that it exists and is active. Nothing in the spec prevents alternating between these states.  
One possible reason for this flexibility is to let recipients pause purchasing (to adjust items or timing) and reopen later without creating a new registry.

### Q4. Registry deletion
In practice, this usually doesn’t matter in terms of functionality: closing a registry is enough to end its public visibility while preserving purchase history (useful for thank-yous and records). Adding deletion would complicate things because purchases would need careful handling to avoid breaking tracking. Unless there’s a policy or legal requirement to remove data, keeping closed registries is the simpler, safer design. If storage or retention costs are a real constraint, a separate data-retention policy could justify adding `deleteRegistry(registry)`, but that’s an implementation concern and not essential to the core behavior.

### Q5. Queries
- Query 1 (registry owner): “What items were purchased and by whom?” — allows the recipient to see who gave what gifts.
- Query 2 (gift giver): “What items are still available for purchase?” — shows which items still need to be bought and in what quantities.

### Q6. Hiding purchases
I would augment the concept specification to support this by allowing the recipient to delay seeing who bought what until they’re ready. We could add a simple visibility flag on the registry and a toggle action. When the flag is on and the registry is active, owner-facing queries suppress purchaser identities (and purchased counts, if desired). When the registry is closed or the flag is turned off, the owner sees full purchase details.

```text
# Additions to GiftRegistration

state
  a set of Registries with
    ...
    a hidePurchases Flag

actions
  setHidePurchases (registry: Registry, flag: Boolean)
    requires registry exists
    effects set registry.hidePurchases := flag
```

### Q7. Generic types
Using generic `User` and `Item` identities (for example, representing `Item` by a stable ID such as an SKU) is preferable to storing item properties (names, descriptions, prices, etc.). This keeps the GiftRegistration concept independent: the registry only needs to identify which item was requested or purchased and should not depend on or duplicate properties that belong in other concepts. It also improves stability over time, since names, descriptions, and prices can change while the identity remains constant, ensuring historical records continue to reference the correct item even if details are updated elsewhere.
