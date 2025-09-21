# Problem Set 2 — Composing Concepts

## Part 1: Concept Questions

### 1. Contexts in `NonceGeneration`
The contexts are for ensuring the uniqueness of the short strings it generates. In the URL shortener, the context is `shortUrlBase` (e.g., `"bit.ly"` or `"tinyurl.com"`). This allows different domains to reuse the same nonce safely. For instance, `bit.ly/abc123` and `tinyurl.com/abc123` can coexist because they belong to different contexts.

### 2. Storing Used Strings
`NonceGeneration` must store used strings to satisfy its principle that every generated string is unique within a context.

With a counter implementation, the abstraction function would be:

Concrete state: `counter = n`  
Abstract state: `used = {counter_to_string(0), counter_to_string(1), ..., counter_to_string(n-1)}`  

The set of used strings represents all nonces that have ever been generated for that context. The counter is just one way to implement this where it implicitly represents this set by knowing that all values from 0 to n-1 have been used, ensuring no duplicates.

### 3. Words as Nonces
| Perspective | Advantage | Disadvantage |
|-------------|-----------|-------------|
| User | Easier to read, remember, and share verbally | Limited pool of words leads to faster exhaustion |

**Refined Concept:**
```markdown
concept NonceGeneration [Context]
purpose generate unique strings within a context
principle each generate returns a string not returned before for that context
state
  a set of Contexts with
    a used set of Words
    a available set of Words
actions
  supplyWords (context: Context, words: set of Words)
    effect adds words to available for this context
  generate (context: Context): (nonce: Word)
    requires available is non-empty
    effect chooses a word from available, moves it to used, returns it
notes
  - Exhaustion policy: when available becomes empty, the system can (a) call supplyWords,
    or (b) sync to a different nonce concept (e.g., NonceGeneration) to ensure continuity.
```

This removes the need for a manual `initialize` during normal operation while making exhaustion behavior explicit and modular.

## Part 2: Synchronization Questions

### 1. Partial Matching
The first sync `generate` only needs the `shortUrlBase` to determine the context for nonce generation. The second sync `register` needs both arguments because it's handling the complete shortening request (it needs the `targetUrl` to store the mapping and the `shortUrlBase` to know where to register it).

### 2. Omitting Names
The convention can only be used when argument and variable names are exactly the same. When they differ, explicit naming is required to avoid ambiguity about which variable maps to which parameter.

### 3. Inclusion of Request
The first two syncs are triggered by user requests (i.e. someone wants to shorten a URL), so they include `Request.shortenUrl`. The third sync `setExpiry` is triggered by an internal system event (i.e. a URL registration completing), not a user request, so no request action is needed.

### 4. Fixed Domain
If the domain is fixed, we can hardcode it in `context` for the `generate` sync and `shortUrlBase` for `register` (i.e. "bit.ly"):
```markdown
sync generate
when Request.shortenUrl (targetUrl)
then NonceGeneration.generate (context: "bit.ly")
sync register
when
  Request.shortenUrl (targetUrl)
  NonceGeneration.generate (): (nonce)
then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl)
```

### 5. Expiry Sync
```markdown
sync expire
when ExpiringResource.expireResource(): (resource: shortUrl)
then UrlShortening.delete (shortUrl)
```

## Part 3: Extending the Design

### Concept: `UserAuth`
```markdown
concept UserAuth [User]
purpose identify users so they can access personalized services
principle after registering and logging in, users can perform authenticated actions
state
  a set of Users with
    a username String
    a password String
  a set of Sessions with
    a token String
    a user User
actions
  register (username, password: String): (user: User)
  login (username, password: String): (session: Session)
  authenticate (token: String): (user: User)
  logout (session: Session)
    requires session exists
    effect invalidates the session so it can no longer authenticate
```

### Concept: `Analytics`
```markdown
concept Analytics [User, Resource]
purpose allow owners to track how often their resources are accessed
principle after creating analytics for a resource, each access increments a count that only the owner can see
state
  a set of Resources with
    an owner User
    an accessCount Number
actions
  createAnalytics (owner: User, resource: Resource)
    requires resource not already tracked
    effect initializes accessCount to 0
  recordAccess (resource: Resource)
    requires resource exists in Analytics state
    effect increments accessCount
  getAnalytics (requester: User, resource: Resource): (count: Number)
    requires requester = owner of resource
    effect returns accessCount
```

### Synchronizations
```markdown
sync createAnalytics
when
  Request.shortenUrl (targetUrl, authToken)
  UserAuth.authenticate (token: authToken): (user)
  UrlShortening.register (): (shortUrl)
then Analytics.createAnalytics (owner: user, resource: shortUrl)

sync trackAccess
when UrlShortening.lookup (shortUrl)
then Analytics.recordAccess (resource: shortUrl)

sync viewAnalytics
when
  Request.viewAnalytics (shortUrl, authToken)
  UserAuth.authenticate (token: authToken): (user)
then Analytics.getAnalytics (requester: user, resource: shortUrl)
```

## Feature Request Analysis

| **Feature** | **Response** |
|-------------|-------------|
| **a) Allowing users to choose their own short URLs** | This can be implemented by adding an optional `customSuffix` parameter to `Request.shortenUrl`. The `register` sync would then first check if the custom suffix is available before using `NonceGeneration`. This is a minimal change since it only adds a conditional branch to the existing sync logic. |
| **b) Using "word as nonce" strategy** | This feature can be supported by replacing the existing `NonceGeneration` concept with a `WordNonceGeneration` concept that uses dictionary words as nonces. Because the interface remains the same, no changes to the syncs are needed. This demonstrates strong modularity, as the implementation can be swapped out without affecting the rest of the design. |
| **c) Including target URL in analytics** | To support grouping by target URL, the `Analytics` state can be extended to store `targetUrl` along with each resource. The `createAnalytics` sync would then pass the `targetUrl` as part of initialization. This change is small and localized to the `Analytics` concept. |
| **d) Generate non-guessable short URLs** | This can be achieved by modifying `NonceGeneration` to use a cryptographically secure random generator rather than a sequential counter. No sync modifications are necessary, because the interface is unchanged. This shows excellent modularity: the improvement is entirely internal to the concept. |
| **e) Supporting analytics for non-registered users** | This feature would break the privacy assumption that analytics are only viewable by the creator. Supporting it would require significant restructuring, such as making analytics public (which is a security concern) or adding a separate concept to handle anonymous tracking. Given these trade-offs, the feature should not be included because it conflicts with the privacy-focused design. |
