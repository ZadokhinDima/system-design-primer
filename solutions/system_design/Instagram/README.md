# Design the Instagram feed, post, like and comment functions

## Step 1: Outline use cases and constraints

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

![Initial design](https://github.com/ZadokhinDima/system-design-primer/blob/master/solutions/system_design/Instagram/instagram_high_level.png?raw=true)

## Step 3: Design core components

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

![Scalled design](https://github.com/ZadokhinDima/system-design-primer/blob/master/solutions/system_design/Instagram/instagram_high_level_scalled.png?raw=true)


We'll introduce some components to complete the design and to address scalability issues.  Internal load balancers are not shown to reduce clutter.

Changes:
* Web-Service and Application Layers are scalled horyzontally. This will allow system to adapt to any load.
* **Main Storage** is scalled using master-slave replication. This also unlocks the full potential of CQRS usage (read/write separation).
* **User service** is removed as it's only responcibility was to get list of subscribers for user.  
    * Potentially user service is required for the system but not for the given use cases.
* **Feed Cache** is also replicated for higher throughput.
* **CDN** added for faster conntent delivery.

Additional considerations:
* Keep only several hundred posts for each user in the **Feed Cache**
* Keep data in **Feed Cache** only for active users.
* At any moment **Feed Service** can update user's feed.
    * Will happen when additional posts requested or user becomes active.
    * **Feed Service** uses data from SQL read replicas.

We should also consider moving some data to a **NoSQL Database**.
