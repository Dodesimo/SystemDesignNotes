
Networking Core Concepts:
- Mostly pick HTTP over TCP.
- WebSockets and Server-Sent Events: when we need real time updates. 
- Server-Sent Events: one-way server to client push.
- Websockets: clients need to push data back to the server. 
- Stateful connection: can’t just use a load balancer, think about connection persistence, what happens when a server load goes down.
- Use RPC internally because it uses binary serialization and HTTP2, making it faster than JSON.
- Load balancing: Layer 7 operates at the application level, routing based on content. 
- Layer 4: operate at the TCP layer and then operate simply based on packets.
API Design:
- Large requests: use pagination (cursor-based when new items get added frequently, offset is great for most canses).
- Use JWT tokens for user session authentication and API keys for service calls. 
Databases:
- Normalization: split data across to avoid duplication, but you may need complex joins to get complete data.
- Denormalization: duplicate data to avoid joins and make reads faster. 
	- Bad if frequent updates, we can use if read-heavy system. 
Database Indexing:
- Indexes make queries fast.
- Most common: a B-tree that keeps data assorted in a way that supports range queries and lookups. 
- There exists specialized indices: full-text indexes for searching and geospatial indexes for location queries.
- Composite queries: have compound indices on both. 
- Specialized needs: thing systems like Elastic search for full-text search, or PostGIS for geospatial queries, eventually sync with primary database through data capture. 
Caching:
- Store frequently accessed data.
- 90% of the time: we use cache-aside with Redis.
- On a read, we check cache first, else we store result in the cache with a TTL and return.
- How to do invalidation?
- Invalidate cache after writes, use short TTLs or combine both. 
Sharding:
- Split data across multiple independent servers (hit storage limits, write throughput limits, or read throughput replicas can’t handle). 
- Shard key: determines how data is sharded.
- Most systems have hash-based sharding, hash the shard key and then modulo to pick a shard (even data distribution). 
- Range based sharding: access patterns have naturally partitioned. 
- Sharding should be optimization only justified when necessary. 
Consistent Hashing: 
- When a simple hash is used to pick what server stores data, adding or removing server changes N, so each key maps to a different server so you have to move data around.
- A problem.
- Consistent hashing: place servers and keys on a virtual ring, hash each key put on the ring, find appropriate server by going clockwise. 
- So when new server is added, keys between server and previous move.
- Remove a server, keys relocate to next server. 
- Talk about this when discussing elastic scaling. How do u handle data or shards.
CAP Theorem:
- Consistency: all nodes will see the same data.
- Availability: all requests get a response.
- Partition tolerance: system works when the network fails. 
- Need to tolerate partition tolerance:
- So if you prioritize consistency, some nodes just refuse to serve requests then provide stale data.
- If you prioritize availability, some nodes will serve requests even during a partition but no guarantee data is right. 
- What situations should we prioritize availability?
	- When we can’t tolerate the app being down, data can be slightly stale.
- Media feeds, recommendation systems, analytics dashboards 
	- What situations can we prioritize consistency:
- When stale data causes problems (inventory, banking, booking systems).
