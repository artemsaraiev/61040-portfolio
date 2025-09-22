### Concept Questions

1.
- **Q: Contexts.** The NonceGeneration concept ensures that the short strings it generates will be unique and not result in conflicts. What are the contexts for, and what will a context end up being in the URL shortening app?
- **A:**  
  A context is just a namespace for uniqueness.
  Context partitions the uniqueness space, so nonces are unique per context but may repeat across contexts. In a shortener, the natural context is the short URL base or domain (e.g., 'https://61040-fa-github.io/'). Using `shortUrlBase` as the context allows the same suffix under different bases and supports user or tenant specific domains.

2.
- **Q: Storing used strings.** Why must the NonceGeneration store sets of used strings? One simple way to implement the NonceGeneration is to maintain a counter for each context and increment it every time the generate action is called. In this case, how is the set of used strings in the specification related to the counter in the implementation? (In abstract data type lingo, this is asking you to describe an abstraction function.)
- **A:**  
  The spec uses a set of used strings to express the freshness invariant abstractly. A counter implementation refines this: let `encode(k)` be the deterministic mapping from integers to nonce strings. If the per-context counter is `n`, then
  - `used(context) = { encode(0), encode(1), ..., encode(n−1) }`
  - `|used(context)| = n` and every element equals `encode(k)` for some `k < n`.  
  This abstraction function shows the counter implementation satisfies the spec's set semantics without explicitly storing the whole set.

3.
- **Q: Words as nonces.** One option for nonce generation is to use common dictionary words (in the style of yellkey.com, for example) resulting in more easily remembered shortenings. What is one advantage and one disadvantage of this scheme, both from the perspective of the user? How would you modify the NonceGeneration concept to realize this idea?
- **A:**  
  Advantage: memorable and easy to type or say aloud.  
  Disadvantage: easier to guess and smaller namespace, so collisions and unavailability rise.  
  **Concept variant:**   
**concept** WordNonceGeneration [Context]   
&nbsp;**purpose** generate memorable word-based nonces within a context   
&nbsp;**principle** each generate returns a word (or word-pair) not returned before for that context   
&nbsp;**state**   
&nbsp;&nbsp;a set of Contexts with   
&nbsp;&nbsp;used set of Strings   
&nbsp;&nbsp;dictionary set of Strings   
&nbsp;&nbsp;blocklist set of Strings   
&nbsp;**actions**   
&nbsp;&nbsp;generate (context: Context): (nonce: String)   
&nbsp;**requires** (dictionary \ blocklist \ used) is nonempty   
&nbsp;**effects** choose nonce $\in$ (dictionary \ blocklist \ used),   
add to used, return nonce

### Synchronization Questions

1.
- **Q: Partial matching.** In the first sync (called generate), the Request.shortenUrl action in the when clause includes the shortUrlBase argument but not the targetUrl argument. In the second sync (called register) both appear. Why is this?
- **A:**  
`generate` only needs the base to pick the NonceGeneration context and produce a suffix; `targetUrl` is not required yet. `register` must create the mapping, so it needs both the base (to form the full short URL) and the `targetUrl`.

2.
- **Q: Omitting names.** The convention that allows names to be omitted when argument or result names are the same as their variable names is convenient and allows for a more succinct specification. Why isn’t this convention used in every case?
- **A:**  
The shorthand only works when the bound variable name equals the parameter or result name and there is no ambiguity. When renaming for clarity, passing multiple values of the same type, or disambiguating across actions in the same `when`, explicit `param: var` avoids collisions and shadowing.

3.
- **Q: Inclusion of request.** Why is the request action included in the first two syncs but not the third one?
- **A:**  
The first two are user initiated flows reacting to `Request.shortenUrl`. The third is a system flow triggered by completion of `UrlShortening.register` and a timer in `ExpiringResource`, so it keys off those completions rather than a new request.

4.
- **Q: Fixed domain.** Suppose the application did not support alternative domain names, and always used a fixed one such as "bit.ly." How would you change the synchronizations to implement this?
- **A:**  
**sync** generate   
**when** Request.shortenUrl (targetUrl)   
**then** NonceGeneration.generate (context: ‘https://bit.ly/’)
**sync** register   
**when**
Request.shortenUrl (targetUrl)
NonceGeneration.generate (): (nonce)
**then** UrlShortening.register (shortUrlSuffix: nonce,
shortUrlBase: ‘https://bit.ly/’,
targetUrl)

5.
- **Q: Adding a sync.** Adding a sync. These synchronizations are not complete; in particular, they don’t do anything when a resource expires. Write a sync for this case, using appropriate actions from the ExpiringResource and URLShortening concepts.
- **A:**  
**sync** expireAndDelete   
**when** ExpiringResource.expireResource (): (resource: shortUrl)   
**then** UrlShortening.delete (shortUrl)

## Extending the design: Analytics

### New concepts

**concept** ShortUrlOwnership [ShortUrl, User]  
**purpose** record which user created which short URL  
**principle** each short URL has exactly one owner; only the owner can view analytics  
**state**  
&nbsp;a set of Ownerships with  
&nbsp;&nbsp;shortUrl ShortUrl  
&nbsp;&nbsp;owner User  
**actions**  
&nbsp;recordOwner (shortUrl: ShortUrl, owner: User)  
&nbsp;&nbsp;requires no ownership exists for shortUrl  
&nbsp;&nbsp;effect create ownership  

&nbsp;authorizeView (shortUrl: ShortUrl, requester: User)  
&nbsp;&nbsp;requires ownership exists for shortUrl and requester = owner  
&nbsp;&nbsp;effect none

---

**concept** UrlAnalytics [ShortUrl]  
**purpose** count number of lookups for each short URL  
**principle** after initialization, every lookup increments exactly one counter  
**state**  
&nbsp;a set of Counters with  
&nbsp;&nbsp;shortUrl ShortUrl  
&nbsp;&nbsp;hits Number  
**actions**  
&nbsp;initCounter (shortUrl: ShortUrl)  
&nbsp;&nbsp;requires no counter exists for shortUrl  
&nbsp;&nbsp;effect create counter with hits := 0  

&nbsp;increment (shortUrl: ShortUrl)  
&nbsp;&nbsp;requires counter exists for shortUrl  
&nbsp;&nbsp;effect set hits := hits + 1  

&nbsp;getHits (shortUrl: ShortUrl): (hits: Number)  
&nbsp;&nbsp;requires counter exists for shortUrl  
&nbsp;&nbsp;effect return hits

---

### Synchronizations

1) **Initialize ownership and analytics on creation**  
&nbsp;**sync** onRegisterInit  
&nbsp;**when**  
&nbsp;&nbsp;Request.shortenUrl (targetUrl, shortUrlBase, creator)  
&nbsp;&nbsp;NonceGeneration.generate (): (nonce)  
&nbsp;&nbsp;UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase, targetUrl): (shortUrl)  
&nbsp;**then**  
&nbsp;&nbsp;ShortUrlOwnership.recordOwner (shortUrl, owner: creator)  
&nbsp;&nbsp;UrlAnalytics.initCounter (shortUrl)

2) **Increment counter on lookup**  
&nbsp;**sync** onLookupIncrement  
&nbsp;**when** UrlShortening.lookup (shortUrl): (targetUrl)  
&nbsp;**then** UrlAnalytics.increment (shortUrl)

3) **Owner requests analytics**  
&nbsp;**sync** viewAnalytics  
&nbsp;**when**  
&nbsp;&nbsp;Request.viewAnalytics (shortUrl, requester)  
&nbsp;&nbsp;ShortUrlOwnership.authorizeView (shortUrl, requester)  
&nbsp;**then** UrlAnalytics.getHits (shortUrl): (hits)

---

### Modularity assessment

1) **Allowing users to choose their own short URLs**  
Realize by adding a custom request and sync that bypasses nonce generation:  
&nbsp;**sync** registerCustom  
&nbsp;**when** Request.shortenUrlCustom (targetUrl, shortUrlBase, shortUrlSuffix, creator)  
&nbsp;**then** UrlShortening.register (shortUrlSuffix, shortUrlBase, targetUrl)  
&nbsp;&nbsp;ShortUrlOwnership.recordOwner (shortUrl: shortUrlBase + shortUrlSuffix, owner: creator)  
&nbsp;&nbsp;UrlAnalytics.initCounter (shortUrl: shortUrlBase + shortUrlSuffix)

2) **Using the "word as nonce" strategy**  
Realize by introducing a `WordNonceGeneration` concept and swapping it into the `generate` sync:  
&nbsp;**sync** generateWordNonce  
&nbsp;**when** Request.shortenUrl (shortUrlBase)  
&nbsp;**then** WordNonceGeneration.generate (context: shortUrlBase)

3) **Including the target URL in analytics**  
Add a grouped counter concept:  
&nbsp;**concept** TargetAnalytics [TargetUrl]  
&nbsp;**purpose** count lookups grouped by target URL  
&nbsp;**state**  
&nbsp;&nbsp;a set of TCounts with  
&nbsp;&nbsp;targetUrl TargetUrl  
&nbsp;&nbsp;hits Number  
&nbsp;**actions**  
&nbsp;&nbsp;increment (targetUrl: TargetUrl)  
&nbsp;&nbsp;**effect** if no entry exists, create with hits := 1; else hits := hits + 1  
&nbsp;&nbsp;getHits (targetUrl: TargetUrl): (hits: Number)  
&nbsp;&nbsp;**requires** entry exists  
&nbsp;&nbsp;**effect** return hits

&nbsp;**sync** onLookupIncrementByTarget  
&nbsp;**when** UrlShortening.lookup (shortUrl): (targetUrl)  
&nbsp;**then** TargetAnalytics.increment (targetUrl)

4) **Generate short URLs that are not easily guessed**  
Add a policy concept that enforces minimum length and entropy:  
&nbsp;**concept** StrongNoncePolicy [Context]  
&nbsp;**purpose** ensure nonces are difficult to guess  
&nbsp;**state**  
&nbsp;&nbsp;minLength Number  
&nbsp;&nbsp;alphabet String  
&nbsp;**actions**  
&nbsp;&nbsp;enforce (nonce: String)  
&nbsp;&nbsp;**requires** |nonce| ≥ minLength and nonce ∈ alphabet*  
&nbsp;&nbsp;**effect** none

&nbsp;Attach this enforcement to both generated and custom suffixes.

5) **Supporting analytics for non-registered creators**  
&nbsp;Undesirable, since it weakens access control. If required, one could issue an ownership token:  

&nbsp;**concept** OwnershipToken [ShortUrl]  
&nbsp;**purpose** allow analytics access without user registration  
&nbsp;**state**  
&nbsp;&nbsp;a set of Tokens with  
&nbsp;&nbsp;shortUrl ShortUrl  
&nbsp;&nbsp;token String  
&nbsp;&nbsp;revoked Flag  
&nbsp;**actions**  
&nbsp;&nbsp;issue (shortUrl: ShortUrl): (token: String)  
&nbsp;&nbsp;**effect** create token with revoked := false; return token  
&nbsp;&nbsp;authorize (shortUrl: ShortUrl, token: String)  
&nbsp;&nbsp;**requires** matching non-revoked token exists  
&nbsp;&nbsp;**effect** none

&nbsp;**sync** viewAnalyticsByToken  
&nbsp;**when**  
&nbsp;&nbsp;Request.viewAnalyticsByToken (shortUrl, token)  
&nbsp;&nbsp;OwnershipToken.authorize (shortUrl, token)  
&nbsp;**then** UrlAnalytics.getHits (shortUrl): (hits)