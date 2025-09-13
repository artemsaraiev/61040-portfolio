## Exercise 1: Reading a concept
1. 
- **Q: Invariants.** What are two invariants of the state? (Hint: one is about aggregation/counts of items, and one relates requests and purchases). Say which one is more important and why; identify the action whose design is most affected by it, and say how it preserves it.

- **A:** 
    - Let `added(i)` be the total quantity ever added for the item `i` via `addItem`. Let `purchased(i)` be $\Sigma$ `purchases.count` for that item. Let `remaining(i)` be the current request's count.   
    **Invariant 1:** `remaining(i) = added(i) − purchased(i)` $\geq$ 0. In other words, the request's count never goes below zero.  
    - **Invariant 2:** every purchase belongs to exactly one request in exactly one registry, and `Purchase.item == Request.item`   
    In the state, each request contains a set of Purchases. That nesting means a purchase is structurally attached to exactly one request (and one registry). Even though Purchase does't carry a request or registry field, the containment provides that linkage.

    - Invariant 2 (linkage) is more important. Without guaranteed Request $\leftrightarrow$ Purchase linkage, you cannot correctly compute `purchased(i)` per registry/item, so `remaining(i)` becomes undefined or wrong. Availability depends on correct linkage.
2.
- **Q: Fixing an action.** Can you identify an action that potentially breaks this important invariant, and say how this might happen? How might this problem be fixed?

- **A:**  
  **Action that can break linkage:** `removeItem(registry, item)`  
  **How it breaks:** If the request has any purchases, deleting it either (a) erases purchase history or (b) orphans those purchases, violating the Request $\leftrightarrow$ Purchase linkage.

  **Fix options:**
  1. **Precondition guard:**  
     Add a requirement:  
     `removeItem(...)` requires request exists **and has zero purchases**
  2. **Archive instead of delete:**  
     Add `request.archived: Flag`.  
     Define `archiveItem(registry, item)` with effects:  
     - `set request.archived := true`  
     - `set request.count := 0`  
     Availability queries then exclude archived requests while purchases remain linked.
  3. **Cancellation flow:**  
     Add `cancelPurchase(purchase)` to reverse counts. Allow `removeItem` only when no active purchases remain.

3.
- **Q: Inferring behavior.** The operational principle describes the typical scenario in which the registry is opened and eventually closed. But a concept specification often allows other scenarios. By reading the specs of the concept actions, say whether a registry can be opened and closed repeatedly. What is a reason to allow this?

- **A:**  
  **Yes.** The preconditions are:
  - `open(registry)` requires the registry exists and `active = false`.
  - `close(registry)` requires the registry exists and `active = true`.
  Nothing forbids calling `open` again after `close`, so the registry can cycle between open and closed.

  **Why allow it:** Practical control. The owner may temporarily close the registry to edit items, prevent purchases during verification or inventory sync, separate phases of an event, or pause for mistake correction, then reopen when ready.

4. 
- **Q: Registry deletion.** There is no action to delete a registry. Would this matter in practice?

- **A:**  
  In practice, this usually doesn't matter. Once a registry is **closed**, it is no longer publicly visible, so givers can't interact with it. Keeping it around is useful for:
  - Returns or exchanges.
  - Historical/audit purposes (analytics, proof of purchase).  

  If deletion were desired (for privacy or clutter reasons), it could be implemented as a soft delete (marking the registry as deleted and hiding it from queries) rather than a hard delete, so purchase history is preserved.

5. 
- **Q: Queries.** What are two common queries likely to be executed against the concept state? (Hint: one is executed by a registry owner, and one by a giver of a gift.)

- **A:**  
  1. **By owner:** "Show me all purchases for my registry" — returns each item, who purchased it, and how many, so the owner can track gifts.
  2. **By a gift giver:** "Show me all available items in this registry" — returns the list of requested items with remaining counts > 0, so the giver knows what is still needed.

6.
- **Q: Hiding purchases.** A common feature of gift registries is to allow the recipient to choose not to see purchases so that an element of surprise is retained. How would you augment the concept specification to support this?

- **A:**  
  Add a flag in each registry, e.g. `ownerCanSeePurchases: Flag`.  
  - Default `false`, so while the registry is open the owner only sees requested items and remaining counts.  
  - Action `setOwnerVisibility` lets the owner toggle this.  
  - Optionally, closing the registry can automatically flip the flag to `true` so full purchase details are revealed afterward.  
  Queries then check the flag: if it's off, hide purchaser identities (or all purchases); if on, show them.

7. 
- **Q: Generic types.** The User and Item types are specified as generic parameters. The Item type might be populated by SKU codes, for example. Explain why this is preferable to representing items with their names, descriptions, prices, etc.

- **A:**  
  Using generic identifiers (like `User` IDs or `Item` SKUs) keeps the concept simple and stable:  
  - **Stability:** Names, prices, and descriptions change; IDs don't.  
  - **Integration:** Works across different stores or catalogs by just mapping IDs.  
  - **Efficiency:** Avoids duplicating bulky, mutable product data inside the registry. SKUs uniquely define an item.  
  - **Flexibility:** Presentation details (name, description, price, images) can be fetched from authoritative catalogs at display time, while the registry only tracks identity and counts.

## Exercise 2: Extending a familiar concept
  **concept** PasswordAuthentication   
  **purpose** limit access to known users   
  **principle**    
  after a user registers with a username and a password,
    they can authenticate with that same username and password
    and be treated each time as the same user   
  **state**   
    &nbsp; a set of Users with …   
  **actions**   
    &nbsp; register (username: String, password: String): (user: User)   
    &nbsp;…   
    &nbsp; authenticate (username: String, password: String): (user: User)   
    &nbsp; …   

   
1.
- **Q:** Complete the definition of the concept state.
- **A:**
**state**  
&nbsp;a set of Users with  
&nbsp;&nbsp;username: String  
&nbsp;&nbsp;salt: Bytes  
&nbsp;&nbsp;passwordHash: Bytes

2.
- **Q:** Write a requires/effects specification for each of the two actions. (Hints: The register action creates and returns a new user. The authenticate action is primarily a guard, and doesn't mutate the state.)

- **A:**  
    **actioins**
    register(username: String, password: String): (user: User)  
    &nbsp;&nbsp;**requires:** no existing user with `username`  
    &nbsp;&nbsp;**effects:** create `user` with  
    &nbsp;&nbsp;&nbsp;&nbsp;- `user.username := username`  
    &nbsp;&nbsp;&nbsp;&nbsp;- `user.salt := RandomBytes()`  
    &nbsp;&nbsp;&nbsp;&nbsp;- `user.passwordHash := H(user.salt || password)`  
    &nbsp;&nbsp;**returns:** `user`

    authenticate(username: String, password: String): (user: User)  
    &nbsp;&nbsp;**requires:** a user `u` exists with `u.username = username` and `H(u.salt || password) = u.passwordHash`  
    &nbsp;&nbsp;**effects:** none (pure guard)  
    &nbsp;&nbsp;**returns:** `u`

*(`H` is a one-way password hashing function; `RandomBytes()` is cryptographically strong.)*

3. 
- **Q:** What essential invariant must hold on the state? How is it preserved?

- **A:**   
    **Invariants:**
    - each `username` is unique
    - for each user `u`, `authenticate(u.username, p)` succeeds **iff** `H(u.salt || p) = u.passwordHash`.  
    
    **Preservation:**  
    - `register` enforces uniqueness and sets the verifier correctly.  
    - `authenticate` does not modify state; it only checks the verifier, so the invariant is preserved.  
4. 
- **Q:** One widely used extension of this concept requires that registration be confirmed by email. Extend the concept to include this functionality. (Hints: you should add (1) an extra result variable to the register action that returns a secret token that (via a sync) will be emailed to the user; (2) a new confirm action that takes a username and a secret token and completes the registration; (3) whatever additional state is needed to support this behavior.)

- **A:**  
  **state**  
  &nbsp;a set of Users with  
  &nbsp;&nbsp;username: String  
  &nbsp;&nbsp;salt: Bytes  
  &nbsp;&nbsp;passwordHash: Bytes  
  &nbsp;&nbsp;confirmed: Flag  

  &nbsp;a set of PendingConfirmations with  
  &nbsp;&nbsp;username: String  
  &nbsp;&nbsp;token: String  

  **actions**  
  register(username: String, password: String): (user: User, token: String)  
  &nbsp;&nbsp;**requires:** no existing user with `username`  
  &nbsp;&nbsp;**effects:** create `user` with  
  &nbsp;&nbsp;&nbsp;&nbsp;- `user.username := username`  
  &nbsp;&nbsp;&nbsp;&nbsp;- `user.salt := RandomBytes()`  
  &nbsp;&nbsp;&nbsp;&nbsp;- `user.passwordHash := H(user.salt || password)`  
  &nbsp;&nbsp;&nbsp;&nbsp;- `user.confirmed := false`  
  &nbsp;&nbsp;create `PendingConfirmations` entry with `(username, token := RandomToken())`  
  &nbsp;&nbsp;**returns:** `(user, token)`  

  confirm(username: String, token: String): (user: User)  
  &nbsp;&nbsp;**requires:** a `PendingConfirmations` entry exists with this `username` and matching `token`  
  &nbsp;&nbsp;**effects:** set `Users[username].confirmed := true`; delete the `PendingConfirmations` entry  
  &nbsp;&nbsp;**returns:** `user`  

  authenticate(username: String, password: String): (user: User)  
  &nbsp;&nbsp;**requires:** a user `u` exists with `u.username = username`, `u.confirmed = true`, and `H(u.salt || password) = u.passwordHash`  
  &nbsp;&nbsp;**effects:** none  
  &nbsp;&nbsp;**returns:** `u`  

## Exercise 3: Comparing concepts
1.
- **Q:** So what exactly is the difference between the standard PasswordAuthentication concept and the PersonalAccessToken concept? Read the Github page carefully, and write a minimal specification of the PersonalAccessToken concept, paying particular attention to the purposes and operational principle. Now consider how the two concepts differ. Finally, say briefly whether you think the GitHub page could be improved, and if so how.    
**Note:** consider only "personal access tokens (classic)" and not "fine-grained personal access tokens."

- **A:**   
    **How it differs from PasswordAuthentication:**  
    - **Creation:** Passwords are chosen by the user; tokens are generated by GitHub (unguessable strings).  
    - **Multiplicity:** A user has one password but may create many tokens (and revoke them individually).  
    - **Revocability:** Tokens can be revoked without changing the account password.  
    - **Usage context:** Tokens are optimized for automated access (scripts, CI, APIs), whereas passwords are optimized for interactive login.  
    - **Identity:** A password is paired with a username to authenticate; a token alone is sufficient, since it both identifies its owner and proves access.  
    - **Persistence:** Tokens are stored (e.g., in a secret manager); users aren't expected to memorize them.  
    
    **GitHub documentation improvement:**  
    The page currently says "Treat tokens like passwords," which blurs the distinction. It could be clearer by stating upfront:  
    - Passwords = one per account, memorized, always paired with username.  
    - Personal access tokens = many per account, generated, revocable, and used *in place of **both username and password*** in many contexts.  
    This would help newcomers understand why tokens exist and when to use them.

    **concept** PersonalAccessToken  
    **purpose** allow access to GitHub services without using the account password, especially for scripts, command-line tools, and integrations  
    **principle**  
    a user can generate a random token string bound to their account;  
    they can use this token in place of the usual username+password pair for authentication in supported contexts (HTTPS Git operations, API calls);  
    tokens can be revoked at any time by the user without affecting the account password.  

    **state**  
    &nbsp;a set of Tokens with  
    &nbsp;&nbsp;owner: User  
    &nbsp;&nbsp;value: String (random, secret)  
    &nbsp;&nbsp;createdAt: Time  
    &nbsp;&nbsp;revoked: Flag  

    **actions**  
    createToken(owner: User): (token: Token)  
    &nbsp;&nbsp;**requires:** user exists  
    &nbsp;&nbsp;**effects:** generate a new random token, store with `owner`, `revoked := false`  
    &nbsp;&nbsp;**returns:** token  

    revokeToken(token: Token)  
    &nbsp;&nbsp;**requires:** token exists and not revoked  
    &nbsp;&nbsp;**effects:** set `revoked := true`  

    authenticate(tokenValue: String): (user: User)  
    &nbsp;&nbsp;**requires:** there exists a non-revoked token with `value = tokenValue`  
    &nbsp;&nbsp;**effects:** none (guard)  
    &nbsp;&nbsp;**returns:** the token's `owner`  

## Exercise 4: Defining familiar Concepts

### 1. URL Shortener

**concept** URLShortener  
**purpose** provide short, convenient aliases for long URLs  
**principle**  
a user can submit a long URL and obtain a short code;  
this code can be user-chosen or auto-generated;  
requests to the short URL redirect to the original long URL;  
codes can be revoked if necessary.  

**state**  
&nbsp;a set of Mappings with  
&nbsp;&nbsp;shortCode: String  
&nbsp;&nbsp;longURL: String  
&nbsp;&nbsp;owner: User  
&nbsp;&nbsp;active: Flag  

**actions**  
createMapping(owner: User, longURL: String, shortCode?: String): (mapping: Mapping)  
&nbsp;&nbsp;**requires:** if `shortCode` provided, it must not already exist  
&nbsp;&nbsp;**effects:** generate unique (if absent) or accept `shortCode`; create active mapping to `longURL`  
&nbsp;&nbsp;**returns:** mapping  

resolve(shortCode: String): (longURL: String)  
&nbsp;&nbsp;**requires:** mapping exists with `active = true`  
&nbsp;&nbsp;**effects:** none  
&nbsp;&nbsp;**returns:** longURL  

deactivate(shortCode: String)  
&nbsp;&nbsp;**requires:** mapping exists  
&nbsp;&nbsp;**effects:** set `active := false`  

Autogenerated short codes must be unique. Expired or deactivated codes should not resolve.

---

### 2. Billable Hours Tracking

**concept** BillableHours  
**purpose** record and track billable work sessions for clients/projects  
**principle**  
an employee starts a session by selecting a project and describing the work;  
the system records the start time;  
when the session is ended, the stop time is recorded and duration computed;  
forgotten sessions can be auto-closed or corrected later.  

**state**  
&nbsp;a set of Sessions with  
&nbsp;&nbsp;employee: User  
&nbsp;&nbsp;project: Project  
&nbsp;&nbsp;description: String  
&nbsp;&nbsp;start: Time  
&nbsp;&nbsp;end: Time? (optional)  
&nbsp;&nbsp;closed: Flag  

**actions**  
startSession(employee: User, project: Project, description: String): (session: Session)  
&nbsp;&nbsp;**requires:** employee exists  
&nbsp;&nbsp;**effects:** create session with current `start=now`, `end = null`, `closed = false`  
&nbsp;&nbsp;**returns:** session  

endSession(session: Session)  
&nbsp;&nbsp;**requires:** session exists and `closed = false`  
&nbsp;&nbsp;**effects:** set `end = now`, `closed = true`  

autoClose(session: Session, cutoff: Time)  
&nbsp;&nbsp;**requires:** session exists with `closed = false` and `start < cutoff`  
&nbsp;&nbsp;**effects:** set `end = cutoff`, `closed = true`  

Auto-close handles forgotten sessions. Billing systems can query durations = `end − start`.

---

### 3. Time-Based One-Time Password (TOTP)

**concept** TOTPAuthentication  
**purpose** provide a second authentication factor based on time-varying one-time codes  
**principle**  
a user registers a shared secret with the system (often by scanning a QR code);  
a code is generated from this secret and the current time window;  
the user must present both their password and a valid TOTP code to authenticate;  
codes expire after a short interval.  

**state**  
&nbsp;a set of TOTPSecrets with  
&nbsp;&nbsp;owner: User  
&nbsp;&nbsp;secret: Bytes  
&nbsp;&nbsp;algorithm: String  
&nbsp;&nbsp;digits: Number  
&nbsp;&nbsp;period: Number (seconds)  

**actions**  
registerTOTP(user: User): (secret: Bytes)  
&nbsp;&nbsp;**requires:** user exists and has no TOTP registered  
&nbsp;&nbsp;**effects:** generate and store new secret for user  
&nbsp;&nbsp;**returns:** secret (to be provisioned into authenticator app)  

generateCode(user: User, time: Time): (code: String)  
&nbsp;&nbsp;**requires:** user has TOTP secret  
&nbsp;&nbsp;**effects:** none  
&nbsp;&nbsp;**returns:** code derived from secret and `floor(time / period)`  

authenticate(user: User, password: String, code: String, time: Time): (success: Bool)  
&nbsp;&nbsp;**requires:** password is correct  
&nbsp;&nbsp;**effects:** check whether `code` matches `generateCode(user, time')` for some `time'` within allowed window  
&nbsp;&nbsp;**returns:** true if match, else false  

