- simple easy to reason about
- written in C, executes commands one at a time on a single thread, making it fast and easy to reason about
	- new versions offload I/O and background work to other threads
- redis not ideal when needing durabiliity
	- two persistence modes
	- RDB: takes periodic snapshots 
	- AOF: logs every write, fsyncs once per second by default
		- can fsync each time but this costs speed
- at core:
	- key-value store
	- every object in redis is value stored at string key (value is where the data structure lives)
- infra configs:
	- runs as a single node w/ high availability replica or as cluster
	- cluster: each key hashes to 16,384 slots, each slot is a node
		- sharding: each node has a share of the slots
	- Redis client will cache the slot-to-node mapping so they can compute slot and connect directly
	- nodes share cluster state through gossip protocol
	- every node knows the full slot map
		- wrong node for a key, get a reply pointing to the right one
	- redis replication is asynchronous
		- primary acknowledges write before replica has seen so when primary dies and replica is promoted, last moments of writes vanish
	- cluster mode:
		- client issues READONLY on connection for replica reads
	- non cluster mode:
		- point read traffic directly at the replica
	- no query router in redis that splits requests across nodes and merges results
	- expect data for command to live on one node
	- two keys need to live together, hash tags:
		- `{user:123}:posts` and `{user:123}:likes` land in the same slot
- can handle 100k writes per second
- redis as a cache:
	- cache keys are Redis keys
	- cached values are Redis values
	- key of `product:123` value blob or Redis hash has fields like `name, price, inventoryCount`
	- TTL on each key, guarantees you never read the value of a key after TTL expires
	- Redis rejects writes once memory is full, for caches configure eviction policy
-  redis as a distributed lock:
	- need to maintain consistency during updates
	- need to make sure multiple people aren't doing update/action at once
	- lock is a key everyone agrees on like `lock:concert:343`
		- try to create the key
		- ```
		  SET lock:concert:343 (actual lock) my_token (value) NX (make it succeed only if the key doesn't exist already) EX 30 (30 second expiry on the key) 
		  ```
		* release the lock:
			* delete the key but don't just do it blindly DEL, lock could have expired
			* check if the key holds token you set and delete only then
			* redis will execute the whole script as one command on the thread (atomic)
		* Redis also supports optimistic locking like CAS'ing: WATCH key, run transaction with MULTI/EXEC, transaction aborts if the key changed
		* single-node lock has a failiure mode where replication is async (if primary dies after lock grant, promoted replica would give it away)
			* need some type of consensus but slow client locks this up
			* fencing token: supply ground truth token when doing a write (used to prevent ghost writes)
		* treat Redis lock as efficiency tool ocassionally failing, not correctness guarantee
* redis for leaderboards:
	* natural fit for leaderboard applications b/c ordered data can be queried in log time
	* high write throughput and low read latency
	* maintain a sorted set for a list of top liked posts
		* ```
		  ZADD tiger_posts 500 "SomeId1" #500 likes for one
		  ZADD tiger_posts 1 "SomeId2" #1 like for another
		  ZREMRANGEBYRANK tiger_posts 0 -6 # remove everything but the top 5 posts 
		  ```
	* we add scores and then only maintain the top 5
* redis for rate limiting:
	* common algo: fixed window rate limiter, number of requests doesn't exceed N over some fixed window of time W
	* request comes in, count exceeds N, reject request otherwise proceed
	* set expiry only when INCR returns 1
		* first request of the window (so after 60 seconds or wtvr)
		* this means that after 60 seconds the key gets deleted and then the key inserted is new
	* sliding window:
		* sorted set per user w/ the request timestamp as the score
		* use `ZREMRANGEBYSCORE`: drop entires older than the window
		* then use `ZCARD` to count what's left
		* count under N, `ZADD` new request
		* make these operations as a Lua script for atomic execution
* redis natively supports geospatial indices:
	* `GEOADD` and `GEOSEARCH`
		* ```
		  GEOADD key longtiude latitude member # adds "member" to index at key
		  GEOSEARCH key FROMLONLAT longitude latitude BYRADIUS radius unit # get all items within radius from the longitutde/latitude
		  ```
		* N + log(M) where N is the number of elements inside the grid bounding box and M is the # of items within the radius
			* first pass grabs N candidates in the box
			* second pass filters them to M items within the exact radius
* redis for event sourcing:
	* redis streams are append-only logs similar to Kafka topics
	* building block for event-sourced designs, store an ordered log of events and get state from it
	* work queue: we add items onto the queue with `XADD` and attach single consumer group to the stream for workers
	* there's a consumer group that tracks which items are pending for each worker
		* worker reads items w/ `XREADGROUP`, processes it and acknowledges
		* consumer group tracks which items are pending with each worker
		* each pending entry carries idle time, when a worker dies mid-task, idle time keeps climbing and gains priority to be processed
		* worker dies, another worker can claim it with `XCLAIM` or `XAUTOCLAIM` and restart the job
	* when to use Kafka:
		* long retention, replay for many independent consumers, or durable ordered throughput where message is unacceptable to lose
* redis for pub/sub:
	* streams are used for consumers to catch up on what's missed
	* care about delivering to whoever is listening right now, Redis supports publish/subscribe message pattern
		* messages are broadcast to multiple subscribers in real time
	* ```
	  PUBLISH channel message # sends message to all subscribers of 'channel'`
	  SUBSCRIBE channel # listen for messages on 'channel'
	  ```
	* cluster mode, partition the key into slots so you use sharded variants `SPUBLISH/SSUBSCRIBE`
	* Pub/Sub is good for real-time communication, but messages aren't persisted
		* subscriber is offline when message published, miss the message entirely
	* subscriber holds connection to nodes that has channels it cares about
		* so millions of channels don't mean millions of connections
	* delivery is at most once, so for message persistence, delivery guarantee stuff, use dedicated message broker
* creating own Pub/Sub:
	* maintaining key of topic whose value is set of subscriber server addresses
	* issue: 
		* more network hops
			* have to connect to all of the downstream subscriber server addresses
			* three instead of two
		* complexity with dealing with server going down and updating the map
			* requires a heartbeat mechanism
* hot key:
	* load on one server is dramatically higher than the rest of the servers
	* solns:
		* client-side caching that has each app server keep a small in-memory cache of hottest items
			* most reads for hot item don't hit Redis
			* result: serve data that's stale, so have small TTLs on small set of keys
		* key copies:
			*  store same data under several keys
			* copies hash to different slots and land on different nodes
			* reads then give a random suffix
		* read replicas:
			* add read replicas for read capacity
			* cluster clients read from primary by default so replicas absorb load if clients configured for replica reads
			* replicas don't do anything for write-hot stuff
* when not to use Redis:
	* not as a system of record (since there's no persistence guarantees)
	* don't use it when working set can't fit in RAM (memory is expensive)
	* single node operations
	* durable, replaying streams with retention: Kafka
	* 