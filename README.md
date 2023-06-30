
# Level 1 - Single Region with Downtime

**In which we deploy the bare-minimum production-ready application.**

This application consists of a single database instance, single backend
service, which mediates client access to the database, and a single static
asset server which serves HTML, CSS, JS, and media files. The backend can be
thought of as a stateless wrapper around the database, which envorces
authentication and authorization rules. All of these programs run out of a
single data center, or single machine. The network is configured so that the
database instance is only accessible to the backend service.

### Components
- 1 database instance (private)
- 1 backend service (serialized data) (public)
- 1 static asset service (binary data) (public)

### Advantages
- Inexpensive
- Simple

### Disadvantages
- Upgrades require downtime
- Slow service for distant clients
- Failures break service globally

### Decision Points
- How do we schedule downtime for updates?

# Level 2 - Single Load Balancer, Multi-Instance Backend

**In which we solve our downtime problem and prevent bad releases from
affecting our entire user base.**

This application adds a second instance of the backend service, which can
respond to reqests if the other fails. We also introduce a single load
balancer, which receives all incoming requests, and determines which backend
service to send traffic to. It can use blue-green / canary style deployments to
eliminate downtime and minimize client exposure to buggy releases.

### Components
- 1 database instance (private)
- 2 backend service (serialized data) (private)
- 2 static asset service (binary data) (private)
- 1 load balancer (public)

### Advantages
- Hypothetically zero-downtime
- New releases can be tested against small audiences
- Protection from single-location failures* (if the load balancer is in a third location)

### Disadvantages
- Blue & Green services must share one version of the database
- Backwards compatibility becomes necessary
- latency increased by added network jump: load balancer -> backend -> db
- Response times are still bottlenecked by clients' distance to the load balancer
- Load balancer is a new fatal point of failure

_🤔 Notice that there's a tradeoff relationship associated with the distance of
the load balancers to the services they balance. Too close, and they may be
affected by the outages that take down their children. Too distant, and
response time becomes non-trivial._

_🤔 How do we protect the load balancer from going down? How do deploy updates
to it?_

_🤔 Notice that the load balancing wraps the parts of our application which are
the most stateless and most prone to frequent changes._

# Level 3 - Regional Load Balancers with CDN and In-Memory Caching

**In which we minimize latency for distant clients and reduce the load on our
database.**

Static binary files are now cached by a CDN, which keeps copies close to clients,
so that they don't need to be retrieved from their original location.

Instances of the backend service which mediate access to the database are now
more widely distributed, and use a nearby in-memory cache (like Redis or
Memcached) to minimize trips to the database.

Every region now has its own load balancer and green / blue pair. The deploy
process would deploy to every region's idle environment. DNS configuration
determines which load balancer clients are directed to when they make a request
to their desired URL. Consult a [list of DNS routing
strategies](./DNS_STRATEGIES.md) for details.

### Advantages
- Drastically reduces latency for read-only activity
- Dramatically reduces the volume of requests sent to the database
- Traffic to failing regions can fall back to the next closest region
- Allows progressive canary rollout by region

### Disadvantages
- Infrastructure costs and complexity jump non-trivially
- Monitoring and debugging become more challenging

_🤔 Do we need a designated cache database like Redis, or is okay to just use a
HashMap-like structure within our backend codebase?_

_🤔 What strategies do we use to prevent our caches from serving stale data?
We could make the cached values have an expiration date, or we could make the
database send notifications to every region when a potentially staling write occurs.
Consider consulting a [more complete list of strategies](./CACHE_STRATEGIES.md)._

_🤔 Do we need to update our deployment process to account for CDN-busting?_

_🤔 Do we cache the results of database queries, or the results of our clients'
HTTP requests?_

_👀 Would others be interested in seeing a talk on [HTTP cache-control headers](./CACHE_CONTROL_HEADERS.md)?_


# Level 4 - Database Replication

**In which we increase the fault tolerance and minimum latency.**

One region is designated as the "leader" instance, and associated with a
particular region. Every other region is assigned a "replica" database instance
that reads from a stream of mutations to the leader instance.

_Alternatively, we could use a multi-leader or peer-to-peer replication strategy,
depending on what our database ecosystem supports._

### Advantages
- Removes the fatal fragility associated with a single db instance
- Gives every region the smallest possible latency

### Disadvantages
- Introduces the problem of electing a new leader in the event of failure
- Exposes clients to increased possibility for data consistency errors


# Level 5 - Microservices

**In which we separate our application into multiple services, and reduce database sizes.**

Our databases "tables" or "collections" are split into multiple services, each with
their own deployment cycles. Each service can adopt the same multi-region strategy
described previously.

### Advantages
- Reduces the write volume on a single "leader" db instance
- Potentially allows the application to be split into multiple codebases of more manageable size.
- Reduces the storage footprint of single db instances, making them more manageable.

### Disadvantages
- Added complexity and latency for resolving relationships between services
- Added complexity burden for versioning and backwards compatibility
- Requires very careful design and anticipation of latent challenges

# Bonus Level - P2P Decentralization

**In which we persuade our users to assume the burden of hosting their own
instances.**

While this option is not viable for most use-cases, it has been implemented
successfully in some domains, such as in cryptocurrency, gaming, and
file-sharing services.

This approach inevitably contains some amount of unavoidable centralization.

### Advantages
- Nearly or fully eliminates infrastructure costs

### Disadvantages
- Cryptographically protecting sensitive data is uniquely challenging in this paradigm
- Usually involves total transparency, which undermines competitive advantage associated with secrecy


