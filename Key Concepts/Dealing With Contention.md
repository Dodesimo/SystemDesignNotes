- multiple processes complete for the same resource at the same time
- reading and writing from a counter should be atomic
	- do so through a compare-and-set: read value and make write condition on it without it having changed 
- booking:
	- conditional writes:
		- decrement something only if a value is left
	- ```
	  UPDATE concerts
	  SET available_seats = available_seats - 1
	  WHERE concert_id = 'weeknd_tour' and available_seats > 0
	  ```
	* safe under concurrency, database doesn't let two updates change the same row at once
	* also need to create a ticket record (so two writes need to stick together)
	* use a transaction: wrap BEGIN and COMMIT so that database treats them as one unit
	* need to guard the resource people are fighting instead of just the counter
		* have actual tickets be rows instead of just maintaining some counter
	* pessimistic locking:
		* pessimistic locking: acquire locks upfront
			* be pessimistic avoid conflicts, assume they happen and blocking w/ locks
		* FOR UPDATE: locks every row that SELECT returns
			* pick a block, no other transaction claims them
			* this is the lock
		* this is done in application logic
			* multiple things are looked at
			* so conditional UPDATE doesn't work since decision is in application logic between read and write
		* lock: prevents other database connections from accessing the same data until lock is released
		* common issues:
			* locking too much or too long: keep a lock for as small as possible
				* lock entire table, serialize every writer slowing down everything
				* don't do slow operations inside the lock
	* optimistic concurrency control:
		* conflicts are rare, detect them after the fact
		* everyone proceeds and catch the collision at write time
		* requires a row that changes every time row is written
			* read row and note value
			* do a change only if the value is what you read 
			* this is literally compare and swap
			* this is a version number
			* needs to be monotonic increasing
	* write skew: no single row collides, conditional UDPATE, row lock, and version check all go through
	* isolation levels:
		* how much a transaction sees of another's work
		* four levels:
			* `READ UNCOMITTED`: uncommitted changes from other transactions
			* `READ COMMITTED`: see committed changes (default)
			* `REPEATABLE READ`: same data read multiple times within transaction stays consistent
			* `SERIALIZABLE`:  strongest isolation, transactions run after each other
				* only one that catches write skew
				* guarantees that the result is same as if transactions ran one at a time
			* ```
			  BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE
			  select count(*) from on_call
			  where team_id = 'payments'
			  and is_active = true;
			  
			  update on_call
			  set is_active = false
			  where engineer_id = 'alice'
			  commit;
			  ```
			* serializable isn't free: makes database track all reads and writes need to find these conflicts
			* most distributed stores and/or document DBs aren't true serializable
			* avoid by materializing the conflict onto a single row: this would allow FOR UPDATE lock or other mechanism
	* distributed lock:
		* need exclusive access across a transaction
		* FOR UPDATE lock can't hold seat for ten minutes
			* pin connection, stall transaction that needs row
			* lives only inside the transaction so other servers can't see it
		* lock is now data
		* distributed lock: own lifetime, reach for it for exclusive access to span multiple steps or call to another service
			* Redis w/ TTL: 
				* atomic operations w/ automatic expiration
				* SET NX: TTL that creates a lock Redis clears on  its own when the TTL passes (only set if the lock doesn't exist)
				* TTL isn't guarantee of exclusivity: if the holder stalls past TTL due to slow network or something, Redis gives lock to next caller
			* can use the database column
				* ```
				  UPDATE seats
				  SET reserved_by = 'user123', reserved_until = NOW() + INTERVAL '10 minutes' where seat_id = 'A15' AND (reserved_until IS NULL OR reserved_until < NOW());
				  ```
			* expiry lives in the where clause
			* disadvantage: writes slower than a cache
			* advantage: no new infra
			* Zookeeper:
				* coordination services for distributed systems
				* strong consistency guarantees
		* summary:
			* conditional write:
				* sql: WHERE predicate, DynamoDB ConditionExpression, Redis SET NX
			* optimistic concurrency:
				* sql: version column  w/ WHERE version = (CAS), HTTP Etags, etc.
			* pessimistic locking: 
				* sql: SELECT ... FOR UPDATE, mutex/distributed lock elsewhere
			* serializable isolation:
				* sql: isolation lvel serializable, relational
			* distributed lock:
				* sql: reservation row w/ a TTL, Redis SET NX EX
	*  deep dives:
		* deadlocks w/ pessimistic locking:
			* ordered locking: acquire locks in a consistent fashion
			* use database detector to clean up the ones that slip
		* handle ABA problem w/ optimistic concurrency
			*  maintain a version column that increments on every update (monotonically increasing)
			* without a version column, put every field you read into the where clause so the write only happens when the entire row matches
		* handle hot celebrity problem (everyone wants a single resource):
			* reframe problem: have ten identical items and run separate auctions or instead of requiring strong consistency, switch to eventual
			* strong consistency: implement queue-based serialization
				* put all requests for that resource into a dedicated queue that gets processed by a single thread
			* 