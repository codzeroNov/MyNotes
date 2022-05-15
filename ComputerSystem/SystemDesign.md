# System Design

## Destributed Systems Intro

We should keep a system simple, but in the real world we can't handle all the traffic in a single machine.

Basically a distributed system is a group of computers working together and the goal is that all the complexity should be hidden completely from the users.

### Fallacies of Distributed Systems

1. Reliable network
2. Zero latency
3. Infinite bandwith
4. Topology never changes
5. Secure network
6. Only one administrator
7. Zero transport cost

### Distributed System Characteristics

- No shared clock ( clock drift - timer of each computer may get out of sync)
- No shared memory
- Shared resources
- Concurrency and Consistency
- Communication between systems ( thrift )
- Required agreed upon format or protocol

### Distributed System Communication

- Server dicovery
- Server crash mid request
- Server response is lost
- client crashes

### Benefits

- More reliable, fault tolerant.
- Scalability
- Lower latency, increased performance (because of multiple DC around the world)
- Cost effective (a bunch of commodity servers)

## Performance Metrics for Distributed System

### Scalability

- Ability of a system to grow and manage increased traffic
- Increased volume of data or request

### Reliability

- Probability a system will fail during a period of time
- Slightly harder to define than hardware reliability

a common way to measure reliability is **Mean Time Between Failure**

MTBF = (total elapsed time - total down time)/number of failures

### Availability

- Amount of time a system is operational during a period of time
- Poorly designed software requiring downtime for updates is less available

**Availability Calculation**

Availability % = (available time / total time) x 100 %

**Redundancy**

if one replica breaks you can have redundant copy that can take over for it which covers up the overall availablity.

> Reliability vs Availability
>
> - Reliable system is always an available system
> - Availability can be maintained by redundancy, but system may not be reliable
> - Reliable software will be more profitable because providing same service requires less backup resources
> - Requirements will depend on function of this software

### Efficiency

- How well the system performs
- Latency and throughtput often used as metrics

### Manageability

- Speed and difficulty involved with maintaining system
- Observability, how hard to track bugs
- Difficulty of deploying updates
- Want to abstract away infrastructure so product engineers don't have to worry about it

### Latency Key Takeaways

- Avoid network calls whenever possible
- Replicate data across data centers for disaster recovery as well as performance
- Use CDN(content distribution network)s to reduce latency
- Keep frequently accessed data in memory if possible rather than seeking from disk, caching

## Horizontal vs Vertical Scaling

### Vertical Scaling

- Easiest way to scale an application
- Diminishing returns, limits to scalabitiy
- Single point of failure

### Horizontal Scaling

- More complexity up front, but more efficient long term
- Redundancy built in
- Need load balancer to distribute traffic
- Cloud providers make this easier

## System Design Components

### Load Balancer

- Balance incoming traffic to multiple servers
- Software or Hardware based
- Used to improve reliability and scalability of application
- Nginx, HAProxy, F5, Citrix

#### LB Routing Methods

- Round Robin
  - Simplest type of routing
  - Can result in uneven traffic
- Least Connections
  - Routes based on number of client connections to server
  - Useful for chat or streaming applications
- Least Response Time
  - Routes based on how quickly server respond
- IP Hash
  - Routes client to server based on IP
  - Usefult for stateful sessions

#### L4 vs L7

- Layer 4
  - Only has access to TCP and UDP data
  - Faster
  - Lack of information can lead to uneven traffic
  - secure (it can look at the ip addr, if we're getting denial of service attack or bad actors going after application, instead of wasting processing power allowing it through to web server or application, L4 can just toss that request)
- Layer 7
  - Full access to HTTP protocol and data
  - SSL termination
  - Check for authentication
  - Smarter routing options

### Caching

#### Why use caching?

- Improve performance of application
- Save money in long term

#### Speed and Performance

- Reading from memory is much faster than disk, 50-200x faster
- Can serve the same amount of traffic with fewer resources (compared to caching date in local memory)
- Pre-calculate and cache data
- Most apps have far more reads than writes, perfect for caching

#### Caching Layers

- DNS
- CDN
- Application
- Datebase

#### Distributed Cache

- Works same as traditional cache
- Has built-in functionality to replicate data, shard data across servers, and locate proper server for each key( warm-up passive cache before they are used)

#### Cache Eviction

- Preventing stale data
- Caching only most valuable data to save resources
- TTL

  - Set a time period before a cache entry is deleted
  - Used to prevent stale data
- LRU

  - Least Recently Used - Once cache is full, remove last accessed key and add new key
  - Least Frequently Used - Track number of times key is accessed, drop least used when cache is full

#### Caching Strategies

- Cache Aside - most common
- Read Through
- Write Through - maintain high consistency but increase latency in user writes
- Write Back - writing data directly to cache, and cache will sync data source asynchronously

#### Cache Consistency

- How to maintain consistency between database and cache efficiently
- Importance depends on use case

### Database Scaling

Basic Scaling Techniques

- Indexes
- Denormalization
- Connetction pooling
- Caching
- Vertical scaling

#### Vertical Scaling

- Get a bigger server
- Easiest solution when starting out

#### Indexes

- Index based on column
- Speeds up read performance
- Writes and updates become slightly slower
- More storage required for index

#### Denormalization

- Add redundant data to tables to reduce joins
- Boosts read performance
- Slows down writes
- Risk inconsistent data across tables
- Code is harder to write

#### Connection Pooling

- Allow multiple application threads to use same DB connection
- Saves on overhead of independent DB connections

#### Caching

- Not directly related to DB
- Cache sits in front of DB to handle serving content
- Can't cache everything

#### Replication and Partitioning

- Read Replicas
  - Create replica servers to handle reads
  - Master server dedicated only to writes
  - Have to handle making sure new data reaches replicas
  - Fault tolerance & avoid single point failure
- Partitioning ( or sharding)
  - Horizontal partitioning
  - Schema of table stays the same, but split across multiple DBs
  - Downsides - Hot keys, no joins across shards
- Vertical Partition
  - Divide schema of database into separate tables
  - Generally divide by funtionality
  - Best when most data in row isn't need for most queries
