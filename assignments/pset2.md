# Pset 2
## Concept Questions
1. **Contexts**

    The context gives us a boundary around the set of which we want to have no repeats, allowing for multiple subdomains of used strings, rather than one big global set.
    For a URL Shortener, a context could be a URL base, allowing for multiple URL bases to have their own independent sets of used strings.

2. **Storing used strings**

    *NonceGeneration* must store sets of used strings because we can never return the same string more than once for a context, so we must store what we have already returned as a way to verify this.
    To implement using a counter, we would need some encoding that inputs an integer $ n \in \mathbb{N}$ and maps to some string, such that no two integers map to the same string. To generate, we plug in our counter to our encoder, return that string, and increment the counter. In terms of an abstraction function, used set = $ a(j) = \{encode(i) \ | \ 0 \leq i \lt j\}$.

3. **Words as nonces**

    One advantage of this scheme is that, in addition to being easily memorable, the strings can be easily spoken. I can tell someone out loud to go to "yellkey.com/computer" much easier than I can to tell them to go to some string made up of random characters.

    One disadvantage is that available words will run out quickly, so a user may not be able to get the word they want.

    To realize this idea, we would add a dictionary set of strings to our state, giving us:

    **state**

    * A set of Contexts with
        * a used set of Strings
        * a dictionary set of Strings

        
    We would also change the effect/requires of generate to be:
    **requires** There is at least one string in the context's dictionary not in its used set.
    **effect** returns a nonce that is in the context's dictionary and is not already used by this context and adds it to this context’s used set.


## Synchronizations for URL Shortening
1. **Partial matching**

    This is because when we generate, the then clause generates a nonce from a context, so we only need our context as an argument for the when clause (in this case the base URL) to generate a unique string. Later, when we register, we need both of them for the then clause so they both appear as arguments in the when clause.

2. **Omitting names**

    We can not omit names when argument or result names are *not* the same as their variable names because then you woulf be passing an argument called a name into a function that expects a different name. We need to specify the binding explicitly as paramName: variableName.

3.  **Inclusion of request** 

    Because setExpiry is not triggered on user request, it is automatically triggered whenever UrlShortening.register is called, so no need for Request.

4. **Fixed domain**

    For generate, in the then clause, we would make our context the fixed domain. For register, we would not need to pass the arg shortURLBase to the when clause, and in the then clause we do shortUrlBase: "bit.ly" (or whatever our fixed domain is). Set expiry remains unchanged.

5. **Adding a sync**

    **sync** expireAndDelete  
    **when** ExpiringResource.system expireResource(): (resource: shortUrl)  
    **then** UrlShortening.delete(shortUrl)

## Extending the design
1. 
**Concept** Ownership[User, Resource]  
**Purpose** Record who owns a resource  
**Principle** Each resource has exactly one owner until it ownership is removed  
**state**   
* a set of Ownerships with
    * owner User
    * resource Resource    

**actions**  
* assign (resource: Resource, owner: User)
    * **requires** no ownership for resource
    * **effects** assigns owner to be the owner of that resource

* verify (resource: Resource, user: User) : (ok: Boolean)
    * **requires** there is an ownership for this resource
    * **effects** returns True if user owns this resource, false otherwise

* removeOwnership (resource: Resource)
    * **requires** there is an ownership for this resource
    * **effects** removes the ownership mapping from our set of ownerships

**Concept** LinkAnalytics [Resource, User]  
**Purpose** Count accesses to a resource
**Principle** when a resource is created, initialize a counter. Each access increments the counter. When the resource is removed, delete the counter  
**state**
* a set of Counters with
    * a resource Resource
    * a count Integer

**actions**  
* init (resource: Resource)
    * **requires** no counter exists for resource
    * **effects** creates a counter for resource with count = 0

* recordAccess (resource: Resource)
    * **requires** counter exists for resource
    * **effects** increments the count by one for that resource

* readCount (resource: Resource) : (count: Integer)
    * **requires** counter exists for resource
    * **effects** returns the count for this resource

* delete (resource: Resource)
    * **requires** counter exists for resource
    * **effects** removes this resource's counter from the set

2. 

**sync** createShortening  
**when**
Request.shortenUrl(user, targetUrl, shortUrlBase)  
UrlShortening.register(shortUrlBase, targetUrl): (shortUrl)  
**then**
Ownership.assign(resource: shortUrl, owner: user)  
LinkAnalytics.init(resource: shortUrl)  


**sync** translateShortening  
**when** UrlShortening.lookup(shortUrl): (targetUrl)  
**then** LinkAnalytics.recordAccess(resource: shortUrl)  


**sync** viewAnalytics  
**when**
Request.viewAnalytics(user, shortUrl)  
Ownership.verify(resource: shortUrl, user): (ok == true)  
**then** LinkAnalytics.readCount(resource: shortUrl): (count)

3. 
* Allowing users to choose their own short URLs
    * To do this, we could add a new sync called customURL that bypasses the Nonce generation and simply registers the custom URL. We would need to fall back to Nonce generation if the custom URL does not meet the requirements of register.

* Using the “word as nonce” strategy to generate more memorable short URLs
    * Make a new concept called WordNonceGeneration that is the same as NonceGeneration except it adds a dictionary set of words to the state. In the syncs, add a flag in the shorten URL flag that would route to the dictionary word nonce generator instead.

* Including the target URL in analytics, so that lookups of different short URLs can be grouped together when they refer to the same target URL
    * Instead of having one action for reading the count, we would have 2; one by long url, one by short url. We don't have to change our Ownership concept as we just specified resource. In our syncs, when we create a shortURL, we now also make an ownership with the target, and when we access the target, we increment the target as well. Our viewAnalytics would split into 2 syncs, one for short (with our short read action) and one for long.

* Generate short URLs that are not easily guessed;
    * Add a concept RandomNonceGenerator that is similar to NonceGenerator except we include a dictionary set of allowed characters in the state. When we generate, the effect is that we create a string of 15 random allowed characters, repeating if this is already used.

* Supporting reporting of analytics to creators of short URLs who have not registered as user.
    * I would not reccommend including this. We only want the owner to be able to see analytics. Not having them register as a user would make this difficult to do and this would potentially allow anyone to see analytics, going directly against that principle. Having them register as a user is how we know who owns the link.








