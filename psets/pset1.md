# Problem Set 1 —  Reading and Writing Concepts

## Exercise 1: Reading a Concept — GiftRegistration

### Q1. Invariants
- **Invariant 1:** For each request, the remaining amount plus the sum of purchased amounts equals the total requested so far for that request.
- **Invariant 2:** Every Purchase belongs to exactly one Request in exactly one Registry, and for that request `purchase.Item == request.Item`.

The more important invariant is Invariant 1, because if this accounting slips—even briefly—the registry can mislead friends about availability (showing items as available when they aren’t) or allow more to be purchased than was requested. That directly breaks the user-facing purpose of the concept: helping recipients get what they asked for without duplicates. The action most constrained by this is `purchase`, whose precondition requires enough remaining count and whose effect both decrements `request.count` and records the purchase, keeping the equality true.

### Q2. Fixing an action
The `removeItem` action can violate this invariant since if a request is removed after purchases have been recorded for that item, those purchases would no longer be connected to any request, so tracking breaks. The fix is to tighten the precondition so a request can be removed only if it has no purchases.

### Q3. Inferring behavior
Yes, a registry can be opened and closed repeatedly. The `open` action requires that the registry exists and is not active, and the `close` action requires that it exists and is active. Nothing in the spec prevents alternating between these states. One possible reason for this flexibility is to let recipients pause purchasing (to adjust items or timing) and reopen later without creating a new registry.

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


## Exercise 2: Extending a Familiar Concept — PasswordAuthentication

### Q1–Q2. State and action specifications

```text
concept PasswordAuthentication
purpose limit access to known users
principle after a user registers with a username and password, they can authenticate
  with the same credentials and be treated as the same user

state
  a set of Users with
    a username String
    a password String

actions
  register (username: String, password: String): (user: User)
    requires no user exists with this username
    effects create and return a fresh user with that username and password

  authenticate (username: String, password: String): (user: User)
    requires a user u exists with u.username = username and u.password = password
    effects return u
```

### Q3. Essential invariant
The essential invariant that must hold on the state is that usernames are unique (each username maps to exactly one user). The `register` action preserves this invariant by requiring that no user exists with the given username before creating a new user.

### Q4. Extension: email confirmation

We can extend the concept by adding email confirmation using a one-time token returned by `register`. The user confirms via `confirm`, which marks them as confirmed and deletes the token. Authentication continues to require only a matching username and password but if you want to allow only verified users to log in, we can simply add a `confirmed` check to `authenticate`’s precondition.

```text
# Additions/updates to PasswordAuthentication

state
  a set of Users with
    a username String
    a password String
    a confirmed Boolean
  a set of ConfirmTokens with
    a user User
    a secret String

actions
  register (username: String, password: String): (user: User, token: String)
    requires no user exists with this username
    effects create user u with username, password, confirmed = false;
      create token t for u with a fresh secret s; return (u, s)

  confirm (username: String, token: String)
    requires a user u with u.username = username and u.confirmed = false
      and a token t with t.user = u and t.secret = token
    effects set u.confirmed = true; remove token t

  authenticate (username: String, password: String): (user: User)
    requires a user u exists with u.username = username and u.password = password
    effects return u
```

## Exercise 3: Comparing Concepts — PasswordAuthentication vs. PersonalAccessToken

### PersonalAccessToken (classic): minimal concept specification

```text
concept PersonalAccessTokenClassic [User, Scope, Resource]
purpose enable programmatic access on behalf of a user using a bearer token limited by scopes
principle a user generates a token with chosen scopes (and optionally an expiration), then uses
  the token in place of a password for API or HTTPS Git operations; the user can later revoke it

state
  a set of Tokens with
    an owner User
    a secret String
    a set of scopes Scope
    an expiresAt DateTime?   // optional expiration
    an active Flag

actions
  generate (owner: User, scopes: Set[Scope], expiresAt?: DateTime): (token: Token)
    effects create and return a fresh active token for owner with secret, scopes, and optional expiry

  revoke (t: Token)
    requires t is active
    effects set t.active := false

  authenticateWithToken (t: Token, op: Operation, res: Resource): (user: User)
    requires t is active, now < t.expiresAt if set, and t.scopes permit op on res
    effects return t.owner
```

### Differences from Password Authentication
Compared with PasswordAuthentication, personal access tokens are scoped credentials: each token is created with explicit, limited permissions, can include an expiration, and can be revoked on its own. This makes tokens temporary, rotation-friendly secrets suited to automation, whereas a password is a single, long-lived credential that effectively carries the user’s full access and must be changed everywhere if compromised. Users can hold many tokens concurrently for different tools or environments, but typically only one account password.

### Suggested Improvement to GitHub Docs

I would start the Github Docs with a clearer “tokens vs. passwords” section that states the security model up front: tokens are scoped, individually revocable, optionally expiring bearer credentials for programmatic access (CLI/API), while passwords are broad, long-lived credentials for human web login. I’d lead with those conceptual differences—scope, expiration, and revocation—and then add a tight comparison table that answers “when to use a token vs. a password.” Finally, I’d explicitly emphasize that tokens are the default for automation and that passwords should be used only for interactive web login.
