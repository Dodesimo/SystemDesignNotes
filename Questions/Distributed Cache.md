- functional requirements:
	- `set, get, delete` key value pairs
	- users should configure expiration time for key-value pairs
	- data should be evicted based on LRU policy
- nonfunctional requirements:
	- store up to a TB of data and handle a peak of 100k requests per second
	- highly available
	- low latency operations
	- support 1TB of data and 100k requests
- core entities:
	- cache that stores key-value pairs (keys and values)
- API:
	- ```
	  POST /:key
	  {
		  "value": "..."
	  }
	  ```
	* getting
	* ```
	  GET /:key -> {value: "..."}
	  ```
	* delete
	* ```
	  DELETE /:key
	  ```
* high level design:
	* users should set, get, delete key-pairs
		* just have a hash table
		* this is literally the Leetcode LRU cache question
	* configure expiration time for key-value pairs
		* we store tuples of `(value, expiry_timestamp)`
		* `get()`: checks if entry expired before returning
		* `set()`: take optional TTL and calculate expiry timestamp
		* ```
		  get(key):
			  (value, expiry) = data[key]
			  if expiry and currentTime() > expiry (it expired):
				  delete data[key]
				  return null
			return value
			
		  set(key, value, ttl):
			  expiry = currentTime() + ttl if ttl else Null (set the ttl)
			  data[key] = (value, expiry)
		  ```
		* issue w/ this?
			* we only delete keys when they are accessed
			* need a background process to scan and remove expired entries
				* ```
				  cleanup()
				  for key, value in data.items():
					  if value.expiry and current_time > value.expiry:
						  delete data[key]
				  ```
			* have to balance checking entries (CPU usages) and memory efficiency 
	* data needs to be evicted in a LRU policy:
		* this is literally the leetcode problem
		* maintain a hash table and a linked list (doubly)
		* doubly linked list maintains order of access
			* head of list contains most recent used items
			* tail contains least recent used items
			* evict item, remove node right before dummy 
			* its like the left and right: things right after the left node are the oldest and what we would evict
			* things before the right node are the newest
		* accessing an item:
			* look up the node in the hash table
			* remove the node from its current position of the list
			* move it to the front of the list
* deep dives:
	* how do we ensure that the cache is highly available and fault tolerant
		* bad solution, synchronous replication
			* write comes in, all replicas get updated synchronously and we only respond when all replicas have acknowledged the write
			* approach can impact write latency and availability if replica is slow/down
			* we only do this if we need strong consistent guarantees
			* write operations wait for confirmation from all replicas, this adds a lot of latency
			* any replica out, system availability goes down since writes can't go ahead without full consensus
		* good solution, asynchronous replication:
			* update one primary copy, propagate changes to replicas async.
			* write only happens when primary has acknowledged the change (eventually consistent)
			* higher availablility because we can proceed w/ writes even when replicas are down
			* challenges:
				* replicas could have stale data until changes fully propagate
				* writes go through a single node, no need for complex conflict resolution
				* failure recover more complex, track what updates missed when replicas are down
				* primary node, need to promote one replica to new primary
		* good solution, peer to peer replication
			* each node is equal and accept both reads and writes
			* through gossip protocols, changes are propagated
			* provides scalability and availability (not a single point of failiure)
			* gossip protocol ensures changes reach all nodes in the cluster
			* challenges:
				* implementation is more complex (each node maintains connections w/ multiple peers and needs to handle conflict resolution)
	* ensure cache is scalable:
		* distribute data/requests intelligently through sharding
		* need to shard key-value pairs across multiple nodes
		* partitioning: splitting data within single database/system
		* sharding: splitting data across multiple machines/nodes
		* calculate the number of machines we need based on latency and storage calculations
			* then pick the max as that's the bottleneck
	* ensure an even distribution of keys across nodes:
		* use consistent hashing
			* naive hashing results in a lot of data movement when nodes get added/removed
		* arrange both nodes and keys in a circular key space
		* keys map to the clock wise node
		* we don't have to redistribute keys across all nodes, just those that map to that node
	* what happens when there is a hot key in the cache?
		* keys get disproportionately high traffic compared to others
		* such as hot reads/hot writes -> reads are of keys that have high volume of read requests, writes are keys that receive a lot of concurrent write requests
		* solutions:
			* don't just scale up hardware of nodes
			* dedicated hot key cache:
				* separate caching layer: specifically there for handling hot keys
				* key id'd as hot, promoted to this specialized tier (uses more powerful hardware, optimized for throughput)
				* challenges:
					* need monitoring to identify hot keys/mechanisms and promote/demote keys between tiers
					* operational overhead in maintaining separate caching infra
					* system needs to be tuned to determine when keys get promoted or demoted
			*  read replicas:
				* multiple copies of the same data across diff. nodes
				* issues:
					* significant overhead in terms of storage since entire nodes get replicated
					* need to manage replication lag and consistency between primary and replica nodes
					* complexity increases as maintaining + monitoring multiple copies of the same data handing failover and sync across all replicas
			* copies of hot keys:
				* selectively copy only specific keys experiencing high read traffic
				* create copies of hot keys across diff nodes (targeted soln. for handling hot spots)
				* first system monitors key access patterns and determines what are hot keys
				* hot: system creates multiple copies w/ diff suffixes
					* these all get distributed to diff. nodes through consistent hashing
					* reads: randomly choose one of the suffixed keys, spreading read load across multiple nodes
					* writes: system updates all copies to maintain consistency
				* issue:
					* need to maintain all copies in sync when data changes
					* all copies of key need to be changed across nodes
					* can't do it simultaneously or atomic because of complexity
					* but its fine b/c caching systems are eventually consistent
					* overhead in detecting hot key and managing lifecycle of copies
	* hot key that gets written to a lot:
		* write batching:
			* multiple write operations get made as a single atomic update
				* good when the final state matters more than tracking each individual update
			* tradeoff: small delay in write visibility, delay is acceptable given substantial performance benefits
			* issues:
				* tradeoff between delay and write visibility
				* longer batching reduces system load but increases time until writes are visible
				* need to handle failures, replay when batcher fails
				* adds inconsistency in reads, some amount of pending writes in the buffer
			* sharding hot key w/ suffixes:
				* system spreads writes across multiple shards w/ suffix strat
				* hot count key gets split into multiple shards that all have unique suffix from 1 to 10
					* write arrives: system selects one of these shards to update
				* issue:
					* this increases complexity of read operations
					* need to aggregate data from multiple shards
						* increases read latency + resource usage, trading write performance for read performance
					* challenge of maintaining consistency across shards during failure scenarios or rebalancing operations
						* number of shards gotta be chosen correctly (too few doesn't handle load correct but too many makes reads too complex)
					* also wouldn't work if we need to use operations that maintain ordering or atomicity
						* can't be easily decomposed
	* performant cache:
		* distributed: worry about unnecessary network chatter and overhead of network hops
		* batch requests: reduces number of round trips between a client and server
		* consistent hashing:
			* reduces latency since we don't query a central routing service, we directly know what shard to go to 
		* have a established pool of connections for the client
			* client shouldn't create new connection for every request
		* 