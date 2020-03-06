Modern IT landscapes typically consist of a bunch of different microservices. Replacing the monoliths brings us more complexity due to more parts and all their dependencies.

A key aspect for running these systems is the appropriate monitoring with the ability to handle this complexity and to observe system performance. It also needs to understand all different communication forms like REST, gRPC, GraphQL, etc.

In this post we will analyse the performance issues of an existing application. In a follow-up blog post we will then present the solution and the resulting performance improvement.

A demo application ([www.coolboard.fun](https://www.coolboard.fun)) for a GraphQL online course was quickly implemented, set up and running, but ran into performance issues...

![Screenshot coolboard.fun Web  app](images/image7.png "coolboard.fun")

With more and more users, the performance went down faster than expected, resulting in:

*   Page load time > 1 second: even a page took up to 15 seconds!
*   Failing end-to-end browser tests after running into timeouts!

While I was developing the app, I never ran into such issues, so I started wondering

*   What was the main bottleneck? Spoiler alert: there is a subtle side effect caused by rate-limiting.
*   Is there any easy way to fix most (the typical 80%) performance issues quickly?
*   Is there at least some low-hanging fruits, just because sometimes I tend to be lazy?
*   And important in the long-term: can we get insights to make the right decision for changing architecture and building blocks later?

Before searching the root issue, we will need to understand the overall structure of the application, and the building blocks of different (micro)services, their dependencies and how they communicate.

## High-level architecture and services

![](images/imageXX.png)

Web (SPA) -> API Server(BFF, Auth) -> Prisma Server(GraphQL - ORM mapping) -> DB (relational)

The Single-page application (SPA) is running in the browser and connects to [Auth0.com](https://auth0.com/) for authentication and accesses the API Server which provides a specific GraphQL API interface and does authentication handling (aka: backend-for-frontend). It can even be scaled up easily because it does not do session handling there. The authentication is only done by exchanging JWT auth tokens.  
The user management and authentication is done via the separate third-party service, [Auth0.com](https://auth0.com/).

The Prisma Server is an ORM and it provides all usual CRUD operations via GraphQL operations.

## Observation

When I open one board page with only a small number of lists of cards (aka: lanes), then the page loads fast.

But when more pages are opened simultaneously or when there is more load, the performance drops and the page seems to be loading slow-ish!

At least the board’s title with its list names seems to appear quickly because they get loaded first.

![loading board page - first part ](images/image5.png "board-initial")

Then, every list of cards gets loaded and the lanes filled with its cards. Under some load the response times can increase to more than 15 seconds - catastrophical!

![board page has been fully loaded](images/image2.png "board fully loaded")

When you are wondering why the application reacts in this way, we need to mention that

*   the purpose of this application was demonstration of the use of GraphQL,
*   developing and testing the application was done on a local dev machine within a docker environment,
*   there was neither much load nor extra load testing until the app going live, so it was not noticed before, and
*   finally, the deployment to a cloud service was not obviously leading to such a bad performance.

Now, I have some suspicions because I am using the free but limited version of Prisma cloud: The communication between my API-gateway and the Prisma cloud server is somehow throttled and limited (more details later).

But let’s start figuring out how the services communicate with each other by the help of some tools.

## Analysis - Apollo Graph Manager

The easiest way to get some metrics was by activating the built-in tracing-feature for sending query metrics in the Apollo-server: After creating an account and api-key on Apollo Graph Manager at [https://engine.apollographql.com](https://engine.apollographql.com) we can activate tracing in the API-gateway. Furthermore, we just need to turn it on by wrapping the server in our API-gateway:

![Extend server to send metrics to apollo graph manager](images/image13.png "Add Apollo engine")

Every request from the Prisma cloud backend by our API-gateway gets logged - after removing any parameters values.

This will give us some insight on the communication between the website in the browser to the API-gateway.Even while the free version has timely limited logging of only the last 24 hours, it already shows us that we run more than 200 queries, while opening the board page 29 times:

![](images/image10.png)

The response-time of the CardList query is distributed between 400 milliseconds and 14 seconds!

## Finding 1: There are too many GraphQL requests triggered

One root cause may be the limitation or throttling of our free GraphCool/Prisma cloud server:

![](images/image15.png)

Until now, we only get the metrics for GraphQL-requests sent from the browser to the API-gateway.

We will need to dive deeper now. We need to inspect the communication between the API-gateway and Prisma cloud GraphQL server, in order to understand which queries are slow or where the bottleneck is.

## Analysis - APM with tracing

At this point I was looking for an application monitoring tool which is capable of understanding GraphQL. That means which is able to understand and differentiate the GraphQL queries which are all sent as usual POST requests to the same endpoint (e.g. /graphql)  
  
I checked well-known tools on the market, but actually there was only InstanaTM capable of tracing this GraphQL protocol communication.  
Instana™️ also provides end-user-monitoring (EUM) together with tracing the communication of microservices down to database operations, which is an ideal tool for our use case.

We will use Instana™️ running as a SaaS version. Additionally, we will need to run the Instana™️ agent in the same environment as our services. The agent sends the recorded monitoring data to the Instana™️ backend.  
For this demo I can also start it in a local docker environment and start the API-gateway there.  
Compared to the production environment we will get different timings, but that is okay, as we just want to focus on the communication flow for now.

### The setup and high-level architecture for our further analysis:

![](images/imageXXX.png)

Let’s start with enabling end-user-monitoring:  
We will need to define a website in Instana™️, and add this snippet into webpage similar to embedding e.g. Google Analytics:

![](images/tracking-script.png)

Everything will work automatically out of the box, we only need to add setting the name of the page, via injecting a javascript call  
`ineum('page', 'main-page')`
at the end of the webpage.

In order to get full tracing and monitoring in our API-gateway running on Node.js, we only need to run these lines before anything else. This activates code injection, so all requests and responses will get traced automatically!

![](images/image17.png)

Let’s start from the user’s perspective:

Instana™️ provides a “website view” where we can see how our boards page with all its resources gets loaded. After filtering for XHR / Post requests, we already see the necessary requests for boards data:

1.  One request, getting board’s name and its lanes’ titles only
2.  Some extra request for each lane (=card list)

![](images/image4.png)

Although this looks pretty fine (load time below 1 second), the performance gets worse when more users load the board page. We can see that after clicking that button to open the Analytics page to show the backend traces.  

![](images/image16.png)

We can see all specific XHR requests to the API-gateway (at localhost:4000) with the different, varying response times (in the last column):

![](images/image14.png)

We need to dive deeper into one of these traces to figure out how it communicates to the Prisma cloud backend.  
First, when filtering for all calls, we can see that varying response times in the right column again.

That already gives some indication for our issues!

![](images/image12.png)

Then, let’s see what happens in the background by selecting one call. This gives some information and shed some light on the communication of API-gateway and the Prisma cloud backend:  

### Traces for the browser requesting the initial board metadata:

![](images/image1.png)

We find two sequential requests to the Prisma backend, called by the API-gateway one after the another:

1.  First, it is requesting some user information.
2.  In the second request it retrieves the board data from the backend. (In the image above it is selected, so we see the query details on the right side)

### Traces for the browser requesting one lane (=card list) with its cards:

![](images/image3.png)

Here, we also find an extra request - for some user data - (see the details on the right side)!

Finally, even while there are only 6 GraphQL requests by the frontend, we will end up in more than 12 backend calls to Prisma backend!

![](images/image6.png)

![](images/image8.png)

## Finding 2: There are unneeded extra requests by the API-gateway

In the analysis above, we found out that the API-gateway is requesting some unneeded and unexpected extra user data from the database backend (at eu1.prisma.sh), doubling the number of requests.

Quickly running into the rate-limiting causes the varying latency with a maximum response-times of up to 14 seconds as Instana™️ shows us here:

![](images/image11.png)  

## How can we solve this?

We quickly found a performance bottleneck and what is causing that problem: The main goal will be to reduce the overall number of GraphQL requests at the backend.

Obviously, even while the architecture and service structure were fully sufficient for a little demo, the best solution is to migrate to a less limited GraphQL persistence service (e.g. FaunaDB) or hosting a Prisma backend service on our own.

To fix the performance issues, we could even add caching in the API-gateway or collapse all GraphQL queries into one huge GraphQL query, but this means adapting the application.  
Low-hanging fruits: As a quick measure we should get rid of fetching extra user information in each request by adapting our API-gateway server!

## Summary

In order to find performance issues it is necessary to have the right tools: not only to monitor performance but also to analyse it easily and find problems quickly.  
The Apollo Engine helped to get some quick statistics first, but will be limited to GraphQL specific operations only.  
Additionally with Instana™️ we get a bigger detailed picture and we can also find the bottleneck in the communication - via GraphQL and other protocols - of the the whole system.

For this post we used Instana™️ for the detection of the performance issues with only a limited view of only a part of the system. You can imagine how effective this can be when used within the whole production system, monitoring all parts of the whole system, and when you also can use its advanced alerting features!  

When you are interested in more details and even want to try Instana™️, you can run a full-featured [14-days trial](https://www.instana.com/trial/?last_program_channel=Partner&last_program=codecentric&utm_source=codecentric&utm_medium=Website&utm_campaign=Partner_Promotions) version. 
There is also this post about how to install [instana on a kubernetes cluster](https://blog.codecentric.de/2019/10/kubernetes-monitoring-mit-instana-teil-1/)(German).  
You could also request a demo or run a PoC together with the [APM team](https://www.codecentric.de/leistungen/it-acceleration/).

As we now have an idea what the root problem is, we can improve the performance by up to 50% with only little effort. This will be presented in a follow-up blog post.