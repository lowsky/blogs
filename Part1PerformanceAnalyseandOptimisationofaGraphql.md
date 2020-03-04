Modern IT landscapes typically consist of a bunch of different micro services. Replacing the monoliths now brings us complexity by a more parts and all its dependencies.

A key aspect for running these systems is the appropriate monitoring with the ability to handle this complexity and help to observe system performance. It also needs to understand all different communication forms like REST, gprc, GraphQL, etc.

Additionally, this advanced tooling supports with real-time performance analysis for finding and fixing bottlenecks quickly to avoid any bad user impression.

In this post we will analyse the performance issues of an existing application. In a follow-up blog post we will then present the solution and the resulting performance improvement.

A demo application ([coolboard.fun][25]) for a GraphQL online course was quickly up and running, but ran into these performance issues.

![Screenshot coolboard.fun Web  app](images/image7.png "coolboard.fun")

With more and more users, the performance went down faster than expected, resulting in:

*   Page load time > 1 second: even a page took up to 15 seconds!
*   Failing end-to-end browser tests after running into time-outs!

While I was developing the app I never ran into such issues, so I was wondering

*   What was the main bottleneck? Spoiler alarm: There is a subtle side effect caused by rate-limiting.
*   Is there any easy way to fix most (the typical 80%) performance issues quickly?
*   Is there at least some low-hanging-fruits, just because sometimes I tend to be lazy?
*   And important in the long-term: Can we get insights to make the right decision for changing architecture and building blocks later?

Before searching the root issue, we will need to understand the overall structure of the application, and the building blocks of different (micro) services, their dependencies and how they communicate.

## High-level Architecture and Services

![](images/image16.png)[\[a\]][26]

Web (SPA) -> API Server(BFF, Auth) -> Prisma Server(GraphQL - ORM mapping) -> DB (relational)  

The Single-page web application (SPA) is running in the browser and connects to [Auth0.com][27]for authentication and accesses the API Server which provides a specific GraphQL API interface and does authentication handling (aka. backend-for-frontend). It can even be scaled up easily, because there it does not do session handling. The authentication is only done by exchanging JWT auth tokens.  
The user management and authentication is done via the separate third-party service, [Auth0.com][28].

The Prisma Server is an ORM and it provides all usual CRUD operations via GraphQL operations.

## Observation

When I only open one board page with only a little number of lists of cards, then the page loads fast.

But when more pages are opened simultaneously or when there is more load, the performance drops and the page feels loading slowish !

This is what the user sees:

The first impression is the Board’s title and its lists names, which gets loaded relatively fast:

At least by loading the board’s title with its list names the user sees some quick response.

![loading board page - first part ](images/image5.png "board-initial")

  
Then, after some moment every list of cards gets loaded and filled with its cards. Under some load this can get catastrophical, and response times increases to more than 15 seconds!

![board page has been fully loaded](images/image2.png "board fully loaded")

So the user sees that the data are loaded in 2 steps:  
First, the logged-in user, then the board’s title and all titles of the Cardlists.  
Then it requests the data for each Card List.

When you are wondering why the application reacts in this unexpected way, we need to mention that

*   Developing the app was mainly on a local dev machine within docker environment
*   There was no load, nor load test, while the app was developed, so it was not noticed
*   Finally the deployment to a cloud service was not obviously leading to the bad performance.

Now, I have some suspicions, because I am using the free, but limited version of Prisma cloud.  
The communication between my API-gateway and the Prisma cloud Server is somehow throttled and limited.

But let’s figure out how the services communicate with each other by the help of some advanced tools. Let’s see which deeper insights we get.

## Analysis - Apollo Graph Manager

The easiest way to get some metrics was activating the built-in tracing-feature for sending query metrics in the Apollo-server: After creating an account and api-key on Apollo Graph Manager at [https://engine.apollographql.com][29] we can activate tracing in the API-gateway. Additionally, we just need to turn it on by wrapping the server in our API gateway:

![Extend server to send metrics to apollo graph manager](images/image13.png "Add Apollo engine")[carbon source-code.][30]

Every request of data from the Prisma cloud by our API-gateway gets logged.

This will give us some insights of  the communication between the Web site in the browser to the API-gateway.Even while the free version has timely limited logging of only the last 24 hours, it already shows us  that we run more than 200 queries, while opening the board page 29 times:  

![](images/image10.png)

The response-time of the CardList query is distributed between 400 milliseconds and 14 seconds! ![](images/image9.png)

## Finding 1: There are too many GraphQL requests triggered

To better understand which queries are slow or where the bottleneck is, we will need to dive deeper and inspect the communication between the API-gateway and Prisma cloud server.  

One root cause may be the limitation or throttling of our free GraphCool/Prisma cloud server:

![](images/image15.png)

Our web client automatically handles it with running retry-after-failure cycles, but we definitely run into these limitations!

Until now, we only get the metrics for request which were sent to the API-gateway.

Let’s figure out how the API-gateway and the Prisma cloud server communicate.

## Instana™️ Analysis

For further analysis we will use the InstanaTM APM monitoring tool which allows us to get full tracing metrics of all requests from the browser down to the database.

Instana™️ is a modern APM-tool which can be used as a Saas or on-prem version.  
Instana™️ also provides end-user-monitoring (EUM) together with tracing the communication of microservices and down to database operations.

Here, we will use the Instana™️ Saas version and just need to start an agent in our local Docker environment which sends the recorded monitoring data to the Instana™️ backend.

### Local Setup

To use Instana™️, I just add end-user-monitoring and start tracing API-gateway (more later)

Compared to the production environment we will get different timings, but that’s okay, as we just want to focus on the communication flow now.

XXX: this needs to be added somehow here. “There is not much to do with Instana™️, worx OOTB...”

*   How we adapted the application to get the metrics

*   Adapt EUM

*   \-> add to main.html page: Source code ...

*   Adapt BFF Node.js API-gateway
*   \-> add node.js - collector: Source code …
    
    * * *
    

XXX: this needs to be added somehow here ...

### Architecture:

browser -> webapp -> API-gateway (node) -> Prisma server (jvm) -> db (mysql)  
\[image\]

Instana™️ provides a “Website perspective” where we can see how our boards page with all its resources gets loaded. After filtering for XHR / Post requests, we already see the necessary requests for boards data:

1.  One request, getting board’s name and its CardList-IDs only
2.  Some extra request for each List of Cards

![](images/image4.png)

* * *

Although this looks pretty fine (load time below 1 second), the performance gets worse when more users load the board page. We can see that after clicking that button to open the Analytics page to show the backend traces.  

![](images/image16.png)

We can see all specific XHR requests to the API-gateway (at localhost:4000) with the different, varying response times (in the last column):

![](images/image14.png)

We need to dive deeper into one of these traces, to figure out how it communicates to the Prisma cloud backend.  
First, when filtering for all calls, we can see that varying response times in the right column again.

That already gives some indication for our issues!

![](images/image12.png)

Let’s see what happens in the background by selecting one call. This gives some information and deeper insights into the communication of API-gateway and the Prisma cloud backend:  

### Traces for the browser requesting the initial board metadata:

![](images/image1.png)

We find 2 sequential requests to the Prisma backend, called by the API-gateway after each other:

1.  First, it is requesting some user information
2.  In the second request it retrieves the board data from the backend. (In the image above It is selected, so we see the query details on the right side)

### Traces for the browser requesting one Card-list with its cards:

![](images/image17.png)

  
Here, we also find an extra request - for some user data - as we can see in the details on the right side:  

![](images/image3.png)

Finally, even while there are only 6 GraphQL requests by the frontend, we will end up in more than 12 backend calls to Prisma backend!

![](images/image6.png)

![](images/image8.png)

## Finding 2: There are unneeded extra requests by the API-gateway

In the analysis above we found that the API-gateway is requesting some unneeded and unexpected extra user data from the database-backend (at eu1.prisma.sh), doubling the number of requests.

Quickly running into the rate-limiting causes the varying latency with a maximum response-times of up to 14 seconds as Instana™️ shows us here:

![](images/image11.png)  

## How can we solve this?

We quickly found a performance bottleneck and what is causing that problem: The main goal will be to reduce the overall number of GraphQL requests at the backend.

Obviously, even while the architecture and service structure was fully sufficient for a little demo, the best solution will be to migrate to a less limited GraphQL persistence service (e.g. FaunaDB) or hosting a Prisma backend service on our own.

To fix the performance issues, we could even add caching in the API-gateway or collapse all GraphQL queries into one huge GraphQL query, but this means adapting the application.  
Low-hanging-fruits: As a quick measure we should just get rid of fetching extra user information in each request by adapting our API-gateway server!

## Summary

To find performance issues it is necessary to have the right tools to not only monitor performance but also to analyse it easily and find problems quickly.  
The Apollo Engine helped to get some quick statistics first, but will be limited to the graphql specific operations only.  
With Instana™️ we additionally get the detailed bigger picture and can also find the bottleneck in the communication patterns of the the whole system.

We used Instana™️ for the detection of the performance issues with only a limited view of only a part of the system.  
You can imagine how effective this could be when it is used within the production system, monitoring all parts of the whole system, and when you also can use its advanced alerting features!  

When you are interested in more details and even want to try-out Instana™️, you can run a full-featured, 14-days trial version: [Instana™️ Trial][31]  
You could also request a demo or run a PoC together the [APM team][32] (German)

There is also this post about how to install [instana on a kubernetes cluster][33]. (German)  

As we now have an idea what the root problem is, we can improve the performance by up to 50% with only little effort. This will be presented in a follow-up blog post.[\[b\]][34]

[\[a\]][35]Different image

[\[b\]][36]In a follow-up blog post we will also look into

\* How we improved the application

GraphQL resolver,

\* How we removed auth-check and its DB request

\* How the performance increased in comparison to old version

A <-> B

[1]: #h.k53h1w73n0yc
[2]: #h.k53h1w73n0yc
[3]: #h.s90a35f3c64u
[4]: #h.s90a35f3c64u
[5]: #h.ahid7djx8ix1
[6]: #h.ahid7djx8ix1
[7]: #h.l7dhpvjokrji
[8]: #h.l7dhpvjokrji
[9]: #h.59y8f6pc0uu7
[10]: #h.59y8f6pc0uu7
[11]: #h.map2iz5gujds
[12]: #h.map2iz5gujds
[13]: #h.3am9dm31mdgg
[14]: #h.3am9dm31mdgg
[15]: #h.2yys5ah0dyy
[16]: #h.2yys5ah0dyy
[17]: #h.ywmamz4gigmm
[18]: #h.ywmamz4gigmm
[19]: #h.rfivyhlmh1kf
[20]: #h.rfivyhlmh1kf
[21]: #h.p20bk4535ah4
[22]: #h.p20bk4535ah4
[23]: #h.57yfhgvupx9y
[24]: #h.57yfhgvupx9y
[25]: https://www.google.com/url?q=https://coolboard.fun&sa=D&ust=1583327294767000
[26]: #cmnt1
[27]: https://www.google.com/url?q=https://auth0.com/&sa=D&ust=1583327294769000
[28]: https://www.google.com/url?q=https://auth0.com/&sa=D&ust=1583327294769000
[29]: https://www.google.com/url?q=https://engine.apollographql.com&sa=D&ust=1583327294772000
[30]: https://www.google.com/url?q=https://carbon.now.sh/?bg%3Drgba(255%252C255%252C255%252C1)%26t%3Done-light%26wt%3Dnone%26l%3Djavascript%26ds%3Dtrue%26dsyoff%3D20px%26dsblur%3D68px%26wc%3Dfalse%26wa%3Dtrue%26pv%3D48px%26ph%3D32px%26ln%3Dfalse%26fl%3D1%26fm%3DHack%26fs%3D14px%26lh%3D133%2525%26si%3Dfalse%26es%3D2x%26wm%3Dfalse%26code%3Dconst%252520server%252520%25253D%252520new%252520GraphQLServer(%25257B%252520schema%252520%25257D)%25253B%25250Aconst%252520httpServer%252520%25253D%252520server.createHttpServer()%25253B%25250Aconst%252520port%252520%25253D%2525204000%25253B%25250A%25250A%25252F%25252F%252520When%252520we%252520have%252520an%252520API%252520key%252520available%25252C%252520wrap%252520it%252520into%252520apollo%252520engine%252520%25250A%25252F%25252F%252520to%252520send%252520tracing%252520metrics%252520to%252520apollo%252520graph%252520manager%25250Aif%252520(process.env.ENGINE_API_KEY)%252520%25257B%25250A%252520%252520%25250A%252520%252520%252520%252520const%252520%25257B%252520ApolloEngine%252520%25257D%252520%25253D%252520require(%2527apollo-engine%2527)%25253B%25250A%252520%252520%252520%252520const%252520engine%252520%25253D%252520new%252520ApolloEngine()%25253B%25250A%25250A%252520%252520%252520%252520engine.listen(%25257B%252520port%25252C%252520httpServer%252520%25257D)%25253B%25250A%25250A%25257D%252520%25250Aelse%252520%25257B%25250A%252520%252520%252520%252520%25252F%25252F%252520start%252520without%252520collecting%252520metrics%25250A%252520%252520%252520%252520httpServer.listen(%25257B%252520port%252520%25257D)%25253B%25250A%25257D&sa=D&ust=1583327294773000
[31]: https://www.google.com/url?q=https://www.instana.com/trial/?last_program_channel%3DPartner%26last_program%3Dcodecentric%26utm_source%3Dcodecentric%26utm_medium%3DWebsite%26utm_campaign%3DPartner_Promotions&sa=D&ust=1583327294781000
[32]: https://www.google.com/url?q=https://www.codecentric.de/leistungen/it-acceleration/&sa=D&ust=1583327294781000
[33]: https://www.google.com/url?q=https://blog.codecentric.de/2019/10/kubernetes-monitoring-mit-instana-teil-1/&sa=D&ust=1583327294782000
[34]: #cmnt2
[35]: #cmnt_ref1
[36]: #cmnt_ref2