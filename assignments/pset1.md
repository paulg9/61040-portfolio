# Pset 1
## Excersize 1
### Questions
1. The two invariants are: 

    a.  Every purchase is linked to one request in the registry for the same item.

    b. For a given item, the amount purchased never exceeds the quntity that was requested. This one I believe is more important as it preserves the 'correctness' of the registry by having no duplicate gifts. This is preserved in the purchase action because it requires at least count (positive requests) and decrements count in the effect.

2. An action that could break this invariant is removeItem. If an item is added, then the item is purchased, then the item is removed, we have a purchased item that is not in the registry. To fix, we can not allow items to be removed if they have been purchased.
3. Yes, a registry can be opened and closed repeatedly. There is nothing in the spec that makes it a one time thing, the only requirements are the registry exists and is active / non active, the only effects are to make it not active / active.
4. In terms of functionality it won't matter as the owner can just close the registry, not allowing any more purchases and hiding it from the public.
5. The registry owner could want to query what items have been purchased and by whom, and the giver of a gift would want to query what items are available.
6. First, to state, I would add a flag under Registry that is true/false if hide purchases is on, call it hidden (default to false). I would also add an action that turns this flag on when we want to enable the feature, call it setHidden(registry). This requires the registry is active. When we query the registry as the owner, if the flag is on, we simply do not return the purchases as stated in the question. 
7. Because this makes the concept design reusable when names of items, for example, change. We want to be able to use the same design for dynamic prices, names, etc.

## Exercise 2
### concept specification:
* **concept** PasswordAuthentication
* **purpose** limit access to known users
* **principle** after a user registers with a username and a password, they can authenticate with that same username and password and be treated each time as the same user
* **state**
    * a set of Users with: 
        * a username String
        * a password String
        * *a confirmed Flag*
        * *a token Secret*
* **actions**
    * register (username: String, password: String): (user: User, *token: Secret*)
        * **requires** no User with username "username" in set of Users
        * **effects** adds a user with username, password, *confirmed=false, generated secret* to set of Users, returns the user *and the secret token to be emailed to the user*
    *  authenticate (username: String, password: String): (user: User)
        * **requires** there exists a user with matching username and password *and confirmed is True*
        * **effects** returns the user
    * *confirm (username: String, token: Secret): (user: User)*
        * ***requires*** *user in users with username==username, confirmed is False, and token==token*
        * ***effects*** *sets confirmed to be True*

### Questions:
1. See above
2. See above
3. The essential invariant is that usernames must be unique. It is preserved in register when we require that username to not be already taken
4. Added specs are italisized, see above. 

## Exercise 3
### concept specification:
* **concept** PersonalAccessToken
* **purpose** authenticate users for CLI/API access 
* **principle** a user generates a token, the token acts on behalf of its owner to authenticate, limited by its scopes and expiration.
* **state**
    * a set of Users with 
        * username String
    * a set of Tokens with
        * an owner User
        * a name String
        * an expiration Date
        * a secret Secret
    * a set of **TokenScopes** with
        * a **token** Token
        * a **scope** Scope
  * a set of **SSOAuthorizations** with
    * a **token** Token
    * an **org** Org

* **actions**
    * generateToken (owner: User, name: String, expiration: Date, scopes: set[Scope]): (token: Token, secret:Secret)
        * **requires** owner exists 
        * **effects** creates a new token with owner, name, expiration, and scopes. return the token and the secret.
    * deleteToken (owner: User, token: Token): ()
        * **requires** tokens owner is owner
        * **effects** remove token and all related TokenScopes and SSOAuthorizations
    * authenticate (username: String, secret: Secret, time: Date, org:Org): (user:User)
        * **requires** User with username, a token in our set of Tokens whose owner is that user and whose secret is the provided secret and is not expired and whose scope meets the requirements of what the user is trying to access. If org is given, then SSOAuthorizations with token, org must exist.
        * **effects** return user
    * **authorizeTokenSSO**(owner: User, token: Token, org: Org): ()
        * **requires** owner of token is owner provided
        * **effects** add SSOAuthorizations(token, org) to set of SSOAuthorizations

    * **revokeSSOAuthorization**(org: Org, token: Token): ()
        * **requires** SSOAuthorizations(token, org) exists 
        * **effects** remove SSOAuthorizations(token, org) from set of SSOAuthorizations

### How is this different than password authentication?
This differs in many ways. First, we can have multiple tokens for a user. Next, tokens come with scopes that specify privelage, passwords are more binary in terms of authentication. Tokens also expire while passwords dont, and can have orginazational SSO authorization.

## Exercise 4

### Billable Hours Tracking
* **concept** BillableHoursTracking
* **purpose** Track billable work by the hour per employee and project
* **principle** an employee starts a work session on a project with a brief description. Time runs until they end the session, and finished sessions are stored.
* **state**
    * a set of ActiveSessions with 
        * a user User
        * a project Project
        * a startTime Date
        * a description String
    * a set of FinishedSessions with
        * a user User
        * a project Project
        * a startTime Date
        * an endTime Date
        * a description String
* **actions**
    * start(user: User, project: Project, description: String, t: Date): (session: ActiveSession)
        * **requires** user exists, prject exists, there are no ActiveSessions with this user.
        * **effects** create & return a new ActiveSession with the given fields
    * end(user: User, actsession: ActiveSession, t: Date): (session: FinishedSession)
        * **requires** actsession exists with user 'user' that was started before given t.
        * **effects** create and return new Session with given fields, remove actsession
    * change(user: User, session: FinishedSession, newEnd: Date): ()
        * **requires** session exists with user 'user' that was started before newEnd
        * **effects** updates session with new endTime
    * discard(user: User, session: ActiveSession)
        * **requires** session exists with user 'user'
        * **effects** removes session

* **invariants**
    * users can hae at most one ActiveSession.
    * Sessions must start before they end.

* **notes**
    * If a user forgot to end, they can change the end date with a provided end date. For the end action, t should be the time when it is called. 
    * If a session is accidentally started, it can be discarded

### Conference Room Booking
* **concept** ConferenceRoomBooking
* **purpose** reserve rooms, avoid conflicts
* **principle** a requester books a room for a time period. The system makes sure that there are no overlapping reservations for the same room. Also, reservations can be changed or canceled.
* **state**
    * a set of Reservations with 
        * a room Room
        * a requester User
        * a start Date
        * an end Date
        * a title String

* **actions**
    * book(user: User, room: Room, start: Time, end: Time, title: String): (reservation: Reservation)
        * **requires** start is before end, there is no conflicting reservation in this room.
        * **effects** create and return the new reservation with the given fields.
    * change(user: User, reservation: Reservation, newStart: Time, newEnd: Time): ()
        * **requires** reservation exists with user 'user'. new start is before new end with no overlapping meetings in the room.
        * **effects** updates start and end times for the reservation
    * cancel(user: User, reservation: Reservation): ()
        * **requires** reservation exists and its user is 'user'.
        * **effects** removes the reservation from the set of reservations.
* **invariants**
    * for any given room, there are no reservations whose times overlap.
* **notes**
    * This allows for bookings of any length at any time, which could be a potential problem in practice.
    

### URL Shortener
* **concept** URLShortening
* **purpose** provide compact, stable links that redirect to longer URLs
* **principle** an owner creates a short link, either with a custom suffix or an auto-generated one, accessing a suffix yields the target URL while the link is active
* **state**
    * a set of Links with 
        * a suffix Suffix
        * a target URL
        * an owner User
        * an active Flag
* **actions**
    * createCustom(owner: User, suffix: Suffix, target: URL): (link: Link)
        * **requires** no Link with this suffix exists.
        * **effects** create and return the new link with the given fields.
    * createAuto(owner: User, target: URL): (link: Link)
        * **requires** 
        * **effects** create unused suffix, create and return the new link with the given fields and created suffix.
    * updateTarget(owner: User, link: Link, newTarget: URL): ()
        * **requires** link exists and its owner is 'owner'.
        * **effects** change the target of the link to the newly provided target.
    * deactivate(owner: User, link: Link): ()
        * **requires** link exists, is active, and its owner is 'owner'.
        * **effects** change active to be False for the link.
    * activate(owner: User, link: Link): ()
        * **requires** link exists, is not active, and its owner is 'owner'.
        * **effects** change active to be True for the link.
    * access(suffix: Suffix): (target: URL)
        * **requires** link exists with suffix, active flag is True.
        * **effects** returns the target of the link.
* **invariants**
    * suffixes are unique
    * cannot access non active links
* **notes**
    * There is no delete, only a safer deactivate. If there are many users, we might run out of good short links after a while.



    
    
