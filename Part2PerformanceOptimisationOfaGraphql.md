# Performance Optimisation of a GraphQL powered App with instana
## Part 2 of Blog post series. -> [Part 1](https://hubs.ly/H0nrMwH0) 

**"Works on my machine."** Okay, but we know quite well software never behaves the same when running on different machines... I knew that, but ran into unexpected performance issues when going live with a simple app.

This is about an existing GraphQL application [www.coolboard.fun](https://www.coolboard.fun) - a kanban board trello-clone app. 
It ran terribly slow when going live, running into performance issues caused by a rate-limited backend.

After the root cause was found (see [previous post](https://hubs.ly/H0nrMwH0)) it's ready to be optimized and we will see the improvements in the results.

Why did I not notice it earlier while developing? I was focused on delivering features, and when testing I was the only user 😉. But with more users the problem emerged 😲!

With appropriate monitoring I was able to find the bottlenecks caused by simple design flaws quickly.

As described in the [previous blog post](https://hubs.ly/H0nrMwH0) in detail, it was caused by a flawed design which was not visible while developing but easily found in production with Instana.

In this post I will describe how the load can easily be reduced by 50% and how the performance can greatly be improved.

### Root cause

As mentioned in the previous post, our Gateway API always fires one additional GraphQL request for user data when the frontend fetches any date.
You might already guess that this could somehow be related to authentication, right? And we will see, that is the right direction ...

---
<!--
To fix it, we need to **understanding** that any user must be **authenticated for read** to any board, and how it will be done:

There are different solutions to handle authentication with GraphQL server[authen..], we use the approach: the client just sends the OAuth JWT authentication token as part of the http header.

This will be avaialable in our GraphQL server in each `resolver` via a context:

So we might ask why the server wants to load some (extra) user data **everythime**?
It is **looking up the user-id** stored in the database together with the auth-token-id.

And even while this works well, it leads to the performance problem when done wrong - our design bug!
-->

### Inefficient Authorisation check

* In GraphQL, the `resolvers` are the "worker" for fetching and providing any piece of data.
* Each query field can have its own resolver method.
* Each resolver method can implement a gate for checking authorisation: e.g. only an admin can "see" everything. 
* For each request there is a context which holds any specific info, e.g. all http request headers.
* A context can also provide access to global services.

In our application, all the resolvers check JWT OAuth token and if a user with that auth-id exists in the database.

In my first implementation the authorization was checked via this helper function `getUserId()`:

```javascript
const getUserId = async (context) => {

   // 1. verify the authentication token (stored in `context`) and retrieve the authenticationID,
  const authenticationID = await retrieveAuthHeaderToken(context)  // 1

  if (authenticationID) {
  
    // 2. lookup the user with this authenticationID,
    const user = await context.db.query.user({where: { authenticationID }}) // 2
    if (user) {

      // 3. retrieve the user's database Id
      return user.id // 3
    }
  }
  
  throw new AuthError()
}
```

And this is how we used this helper in our GraphQL resolvers:

```javascript
// resolvers/query.js

export const resolvers = {

  // get currently logged-in user
  async currentUser(parent, args, context, info) {
  
    const id = await getUserId(context)    // 1
    return context.db.query.user({ where: { id } }, info)
  },

  // get any specific board
  async board(parent, boardId, context, info) {
    await getUserId(context)    // 2
    
    return context.db.query.board({ where: { id: boardId } }, info)
  },

  async cardlist(parent, { where }, context, info) {
    await getUserId(context)    // 2
    
    return context.db.query.list({ where }, info)
  }
}
```

At first view, this implementation seems to be correctly. It is blocking any non authenticated access. 

It worx. Done. Wait! 😯

Can you spot the mistake? 

1. 🧐 The `currentUser()` retrieves the database-id, and fetches the user from the database a second time.
2. 🧐 Without any need for the `user's id` - why do we look-up the user in the database? This was not required at all.

# Authentication-check improved

We introduce extract this method to only verify that the OAuth token from http-header is valid:
```ensureAuth0TokenValid()```

Additionally we do the user lookup in the resolver itself directly, after extracting the authentication ID (part of OAuth token)

The adapted resolvers are now:

```javascript
// resolvers/query.js

export const Query = {
  async currentUser(parent, args, context, info) {
    // checks token from request header, and extracts oauth-id
    const authenticationID = await retrieveAuthHeaderToken(context)
    return await context.db.query.user({where: { authenticationID }})    
  },
  
  async board(parent, boardId, context, info) {
    // checks token from request header
    await ensureAuth0TokenValid(context)
    return ctx.db.query.board({ where: { id: boardId } }, info)
  },

  async cardlist(parent, { where }, context, info) {
    // checks token from request header
    await ensureAuth0TokenValid(context)    
    return ctx.db.query.list({ where }, info)
  }
}
```

There is another simple optimisation possible, because the relation of authentication-id to user-id does not change
at all. 
We can hold that information in a lookup table, but need to load the info once in the lifecycle of the server, - so it is less ideal with serverless lambdas.

```javascript
 // server.js  
const userIdByAuthIdLookup = {};  

export lookupUserWithAuthId = async (authenticationID) => {
    return userIdByAuthIdLookup[authenticationID] ?? 
        userIdByAuthIdLookup[authenticationID] = await db.query.user({where: { authenticationID }})
 }
```

This simplifies the resolver:

```javascript 
  // ...
  async currentUser(parent, args, context, info) {
    const authenticationID = await retrieveAuthHeaderToken(context)
    return lookupUserWithAuthId(authenticationID)
  }
```

As a side-effect this can also be used to optimize our GraphQL mutations which are using the user's id, too:

```javascript
// mutations.js
export const Mutations = {
    async createBoard(parent, { name }, context, info) {
        const authenticationID = await retrieveAuthHeaderToken(context)
        
        const userId = context.lookupUserWithAuthId(authenticationID).id
        
        return ctx.db.mutation.createBoard({ data: { name, createdBy: userId } }, info)
    }
```

 
### Summary:

* We replaced the authentication verification with just the OAuth token verification for query operations
* We removed an unneeded database access for retrieving the data of the currently logged-in user.
* We are now caching the result of the User lookup.

# Verification

## Let's test the result

We want to run a **simple**, but reproducable scenario: 
It will just trigger open the board page 10 times with a 2 seconds delay. This will be "enough" for up to one minute.
Then I will wait one minute, to get out of the rate-limiting time slot, and repeat it again.
With repeating this again, we can ensure to get more solid stats.
-> We will see 3 sections in our diagrams...

```shell
#!/bin/bash 
for i in 1 2 3 4 5 6 7 8 9 10 ; do \
   open https://localhost:3000/board/ck5sc7nis74vk0901gvvr42hi ; \
   sleep 2 ; \
done                                                                     
``` 
and run it 3 times via:

```shell 
sleep 60 ; openBoard10times
sleep 60 ; openBoard10times
sleep 60 ; openBoard10times
```

## Expected result and comparison
After logging-in once into the browser, we are staying authenticated for the testing.
So, effectively, our frontend opens 3 times 10 board pages.

Our frontend will send these GraphQL requests to the API gateway server:

* 30 board queries
* 30 current-user queries
* 150 (=30*5) cards-list queries.


### Overall calls from the API gateway to the GraphQL backend

| Before Optimisation | After Optimisation |
|---|---|
 
![prisma requests](https://dev-to-uploads.s3.amazonaws.com/i/0s2jpkhmjyilobnp2bjp.png)
At the end, we saved almost 50% load on our backend, by reducing the number of requests from > 350 to 180 !

We can go deeper into the numbers for each GraphQL query (type), by grouping them:

![GraphQL Calls](https://dev-to-uploads.s3.amazonaws.com/i/7s4ehsqordu33696o6z7.png)

-> Before optimisation this would lead to **420 calls** in summary:

* 30 user queries + (30 user queries for auth check)
* 30 board queries + (30 user-queries for auth check)
* 150 card-list queries + (150 user-queries for auth check)

    ⚠️ The rate-limiting causes drops and timeouts of about 65 responses!

-> After optimising it results in overall **180 calls** 👍

### latencies

And after optimisation, we had only minimal latency compared to before.

Here, Instana gives us more interesting details: 😎

![latency](https://dev-to-uploads.s3.amazonaws.com/i/aubf51vhsxw2nr9j1sj5.png)

### Page load times for GraphQL requests

And finally the improved page load times show much better results:
(attention, the vertical scale is different)

![Load time on page](https://dev-to-uploads.s3.amazonaws.com/i/7qlfikpdpdu78oo22by6.png)


## Outlook

* The performance improvement shown was just some simple optimisation to reduce the bottleneck.
* We could try to do more optimisation: e.g. why not sharing the userid to the client and send it together with auth-header? That will at the end only save us one extra lookup for some GraphQL operations, but will make the app unsecure, because sharing internal data. (Don't do that!)

-> Finally the natural performance limit - defined by the current backend - is reached. 

This learning allows us to choose an alternative wisely. Hosting the prisma database and running the prisma server by our own will be the cheapest solution.


## Result, Effect
 
We found that improving the performance was not a "big thing". While in this case was hidden while development, it got noticeable in production, caused by a third-party system.

* Such small issues can easily slip through while designing and implementation.
* We used appropriate monitoring and tooling to easy locate it ([part 1](https://hubs.ly/H0nrMwH0)):
* We verified our results after the optimisation was implemented.

Even while this was only a small project, you can imagine how difficult it can get on a bigger system or more distributed systems!
