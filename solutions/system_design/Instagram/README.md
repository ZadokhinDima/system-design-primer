# Design the Instagram feed, post, like and comment functions

## Step 1: Outline use cases and constraints

> Gather requirements and scope the problem.
> Ask questions to clarify use cases and constraints.
> Discuss assumptions.

Without an interviewer to address clarifying questions, we'll define some use cases and constraints.

### Use cases

#### We'll scope the problem to handle only the following use cases

* **User** posts media
    * **Service** saves post to system. Make it available for other user`s feed 
* **User** views the feed
* **User** likes a post
* **User** leaves a comment under post
* **Service** has high availability

#### Out of scope

* **User** posting or viewing instagram stories, using DMs
* Analytics
* Reels
* Ads added to user feed

### Constraints and assumptions

#### State assumptions

General

* Traffic is not evenly distributed
* Eventual consistency: it is ok if post is not immediatelly available to the followers
* 2 billions active users
* 3,5 billion user accounts
* 66_000 posts per minute (1_100 per second)
* 100x more read requests ( 100k/s )
* 10x more likes and 10x comments on average ( 10k/s ) 

Timeline

* Loading the home (recommended feed) should be fast
* More read operations then write operations

#### Calculate usage

* Size per post:
    * `post_id` - 8 bytes
    * `user_id` - 32 bytes
    * `caption` - 140 bytes
    * `media` - 2 MB average
    * Total: ~2MB
* 5,7 PB of new content per month
    * 2 MB per post * 1100 posts per second * 60 * 60 * 24 * 30 days per month
          
## Step 2: Create a high level design

> Outline a high level design with all important components.

![Imgur](http://i.imgur.com/48tEA2j.png)

## Step 3: Design core components

> Dive into details for each core component.

### Use case: User posts a photo

Post data should be stored in relational DB, along with data about likes and comments. There are relations between entities so we can use advantages of relational DBs. Also, likes and comments counters should be available directly in post entity. This breaks normalization but we will not need to peform joins for each post shown in timeline for user. **Main Storage**

Users data and subscription relations also stored in **Main Storage**.

User's recommended feed should be stored separately in **Feed Cache** (Memcached, Redis, etc).

We could store media such as photos or videos on an **Object Store**.

* The **Client** sends post to the **Web Server** (POST request), running as a [reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** contacts the **Post service**, which does the following:
    * Stores the post on a **SQL database**
    * Queries **User Service** to find all users who might be interested in post
    * For those users updates the **Feed Cache** (add this post to their cached feed)
    * Stores media in the **Object Store**

For internal communications, we could use [Remote Procedure Calls](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc).

### Use case: User views the homepage

* The **Client** posts a get recommended posts request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server contacts the **Feed Service**, which does the following:
    * Gets the feed data stored in the **Feed Cache** for users, containing post ids and user ids
        * If data is not available here, feed is build from scratch using data from **Main Storage**
    * Queries the **Main Storage** to obtain additional info about the posts and users who posted.
    * Data is aggregated into one feed response and returned to **Client**

### Use case: User likes or comments a post

* The **Client** posts a user timeline request to the **Web Server**
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** calls **Reaction Service**
* **Reaction Service** builds reaction entity (Like or Comment) and stores it to **Main Storage**

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/jrUBAF7.png)

**Important: Do not simply jump right into the final design from the initial design!**

State you would 1) **Benchmark/Load Test**, 2) **Profile** for bottlenecks 3) address bottlenecks while evaluating alternatives and trade-offs, and 4) repeat.  See [Design a system that scales to millions of users on AWS](../scaling_aws/README.md) as a sample on how to iteratively scale the initial design.

It's important to discuss what bottlenecks you might encounter with the initial design and how you might address each of them.  For example, what issues are addressed by adding a **Load Balancer** with multiple **Web Servers**?  **CDN**?  **Master-Slave Replicas**?  What are the alternatives and **Trade-Offs** for each?

We'll introduce some components to complete the design and to address scalability issues.  Internal load balancers are not shown to reduce clutter.

*To avoid repeating discussions*, refer to the following [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) for main talking points, tradeoffs, and alternatives:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
* [Horizontal scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Web server (reverse proxy)](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [API server (application layer)](https://github.com/donnemartin/system-design-primer#application-layer)
* [Cache](https://github.com/donnemartin/system-design-primer#cache)
* [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL write master-slave failover](https://github.com/donnemartin/system-design-primer#fail-over)
* [Master-slave replication](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Consistency patterns](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [Availability patterns](https://github.com/donnemartin/system-design-primer#availability-patterns)

The **Fanout Service** is a potential bott

Additional optimizations include:

* Keep only several hundred tweets for each home timeline in the **Memory Cache**
* Keep only active users' home timeline info in the **Memory Cache**
    * If a user was not previously active in the past 30 days, we could rebuild the timeline from the **SQL Database**
        * Query the **User Graph Service** to determine who the user is following
        * Get the tweets from the **SQL Database** and add them to the **Memory Cache**
* Store only a month of tweets in the **Tweet Info Service**
* Store only active users in the **User Info Service**
* The **Search Cluster** would likely need to keep the tweets in memory to keep latency low

We'll also want to address the bottleneck with the **SQL Database**.

Although the **Memory Cache** should reduce the load on the database, it is unlikely the **SQL Read Replicas** alone would be enough to handle the cache misses.  We'll probably need to employ additional SQL scaling patterns.

The high volume of writes would overwhelm a single **SQL Write Master-Slave**, also pointing to a need for additional scaling techniques.

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

We should also consider moving some data to a **NoSQL Database**.

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process
