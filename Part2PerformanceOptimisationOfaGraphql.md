# Performance optimization of a GraphQL powered App with instana
## Part 2 of Blog post series. -> [Part 1](https://hubs.ly/H0nrMwH0) 

**"Works on my machine."** Okay, but we know quite well software never behaves the same when running on different machines... I knew that, but ran into unexpected performance issues when going live with a simple app. Here’s how I fixed the problem and improved performance.

This is about an existing GraphQL application [www.coolboard.fun](https://www.coolboard.fun) - a kanban board trello-clone app. 
It ran terribly slow when going live, running into performance issues caused by a rate-limited backend.

After the root cause was found (see [previous post](https://hubs.ly/H0nrMwH0)) it's ready to be optimized and we will see the improvements in the results.

Why did I not notice it earlier while developing? I was focused on delivering features, and when testing I was the only user 😉. But with more users more problems emerged 😲!

With appropriate monitoring I was able to find the bottlenecks caused by simple design flaws quickly.

As described in the [previous blog post](https://hubs.ly/H0nrMwH0) in detail, it was caused by a flawed design which was not visible while developing but easily found in production with Instana.

In this post I will describe how the load can easily be reduced by 50% and how the performance can greatly be improved.

We will remove a bottleneck in the API-Gateway:
![coolboard-on-dockerinstana.png](https://raw.githubusercontent.com/lowsky/blogs/master/images/coolboard-on-dockerinstana.png)

### Root cause

As mentioned in the previous post, our Gateway API always fires one additional GraphQL request for user data when the frontend fetches any data.
You might already guess that this could somehow be related to authentication, right? And we will see, that is the right direction ...

### Inefficient Authorisation check

* In GraphQL, the `resolvers` are the "worker" for fetching and providing any piece of data.
* Each query field can have its own resolver method.
* Each resolver method can implement a gate for checking authorisation: e.g. only an admin can "see" everything. 
* For each request there is a context which holds any specific info, e.g. all http request headers.
* A context can also provide access to global services.

In our application, all the resolvers check JWT OAuth token and if a user with that auth-id exists in the database.

In my first implementation this helper function `getUserId()` checks the authorization: 

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

At first sight, this implementation seems to be correct. It is blocking any non-authenticated access. 

“It worx”... “Done!”... “Wait?!?”... 😯

Can you spot the mistake? 

1. 🧐 The `currentUser()` retrieves the user's id, and loads the user from the database a second time.
2. 🧐 Without any need for the `user's id` - why do we look-up the user in the database? This was not required at all.

# Authentication-check improved

We extract the functionality to only verify that the OAuth token from http-header is valid:
```ensureAuth0TokenValid()```

Then we do the user lookup directly in the resolver itself, after extracting the authentication ID (part of OAuth token).
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

There is another possible simple optimization because the relation of authentication-id to user-id does not change **at all**.
 
We can hold that information in a lookup table, but need to load the info once in the lifecycle of the server - so it is less ideal with serverless lambdas.

```javascript
// server.js  
const userIdByAuthIdLookup = {};  

export const lookupUserWithAuthId = async (authenticationID) => {
    return userIdByAuthIdLookup[authenticationID] ?? 
        (userIdByAuthIdLookup[authenticationID] = await db.query.user({where: { authenticationID }}))
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
}
```

 
### Summary:

* We replaced the authentication verification with just the OAuth token verification for query operations.
* We removed an unneeded database access for retrieving the data of the currently logged-in user.
* We are now caching the result of the User lookup.

# Verification

## Let’s run this **simple** reproducible scenario: 
We will trigger the opening of the board page in the browser 10 times with a 2 seconds delay. This will create "enough load":  the loading will need up to one minute to be finished.

```shell
#!/bin/bash 
for i in 1 2 3 4 5 6 7 8 9 10 ; do \
   open https://localhost:3000/board/ck5sc7nis74vk0901gvvr42hi ; \
   sleep 2 ; \
done                                                                     
``` 

Then I will **wait one minute** to get out of the rate-limiting time slot, and repeat it. After repeating this once again, we can ensure to get **more solid stats**:

```shell 
openBoardPage_10times
sleep 60
openBoardPage_10times
sleep 60
openBoardPage_10times
```

This generates 3 sections we will see in the charts in our results...

## Expected result and comparison

After logging-in once into the browser, we are staying authenticated for the testing.
So, effectively, our frontend opens 3 times 10 board pages.

Our frontend will send these GraphQL requests to the API gateway server:

* 30 board queries
* 30 current-user queries
* 150 (=30*5) cards-list queries.

Our API gateway will send requests to the GraphQL backend
 
-> Before optimization this lead to **420 calls** in summary:

* 30 user queries + (30 user queries for auth check)
* 30 board queries + (30 user-queries for auth check)
* 150 card-list queries + (150 user-queries for auth check)

-> After optimising it results in only **180 calls** 👍

We can see the reduced number of calls, because all responses arrive in a shorter time see (A).

With fewer requests, the number of waiting requests caused by rate-limiting is smaller, and the latency goes down (B).

![app-Calls-Latency-infra](https://raw.githubusercontent.com/lowsky/blogs/master/images/app-Calls-Latency-infra.png)

At the end, we saved ca. 50% load on our backend, by reducing the number of requests from 420 to 180 !

### Latencies

After optimization, the requests sent from the browser have less latency compared with before.

Here, Instana gives us more interesting insights: 😎

The (GraphQL) requests sent from our Website to the API-gateway show how the response times for loading the data on the page goes down by factor 3, overall all pages get loaded in a shorter time.

![websites-Calls-Latency-infra](https://raw.githubusercontent.com/lowsky/blogs/master/images/websites-Calls-Latency-infra.png)

### Page load times for GraphQL requests

And finally the improved page load times reflect the improvement:

![an-websites-httpRequs-Retrieval-Time-](https://raw.githubusercontent.com/lowsky/blogs/master/images/an-websites-httpRequs-Retrieval-Time-.png)

Here we see the result of our performance optimization: much smaller retrieval times!


## Looking forward

* The performance improvement shown was just some simple optimization to reduce the bottleneck.
* We could try to do more optimization: e.g. why not share the userid to the client and send it together with auth-header? That will at the end only save us one extra lookup for some GraphQL operations, but will make the app unsecure, because sharing internal data. (Don't do that!)
* **Finally**, the natural performance limit, defined by the current backend, is reached. 

Our learnings let us think about alternatives more wisely, for example:

* The quick solution: Cut the limit by paying more for Prisma Cloud service.
* We could host the Prisma database and run the Prisma server on our own.
* Affects also the UI frontend: We could also change the API to allow loading the whole board with only one GraphQL request.

## Result, Outcome and Impact
 
We found that, once we understood the issue, improving the performance was not difficult. While the problem was hidden in development, it got noticeable in production, caused by a third-party system.

* Such small issues can easily slip through while designing and implementation.
* We used appropriate monitoring and tooling to easily locate it ([part 1](https://hubs.ly/H0nrMwH0)).
* We verified the results or our optimization.

Even though this was only a small project, you can imagine how difficult troubleshooting can get on a bigger system or more distributed systems!

--- 
_For publishing on the blog post on codecentric pages:_

When you are interested in more details and even want to try Instana™️, you can run a [full-featured 14-days trial version](https://www.instana.com/trial/?last_program_channel=Partner&last_program=codecentric&utm_source=codecentric&utm_medium=Website&utm_campaign=Partner_Promotions). There is also this post about how to install [Instana on a kubernetes cluster](https://blog.codecentric.de/2019/10/kubernetes-monitoring-mit-instana-teil-1/) (German).
You could also [request a demo](https://www.instana.com/schedule-demo/) or run a PoC together with the [APM team](https://www.codecentric.de/leistungen/application-performance-management).