* functional requirements:
	* riders should input start location, destination, get fare estimate
	* riders should request ride based on estimated fare
	* upon request riders should be matched with nearby/available driver
	* drives should be able to accept/decline requests
* nonfunctional requirements:
	- low latency matchmaking (<1 minute)
	- strong consistency in ride matching, prevent driver from getting multiple rides
	- handle throughput w/ peak hours or special events
- core enttities:
	- user/rider: requests rides, includes personal info like name/contact details, preferred payment methods
	- driver: users registered as drivers on platform and provide transportation
		- has personal details, vehicle information, preferences, availability
	- fare: estimated fare for a ride, pickup + destination, estimated fare, and estimated time of arrival
	- ride: individual ride form moment rider confirms fare and requests ride
		- records all details of ride, such as identities of rider and drive, vehicle details, state, and planned route
	- location:
		- real-time location of drives (lat/long), timestamp of last update
		- important for matching riders with nearby drivers
		- tracking ride progress
- API interface:
	- get a fare:
		- POST: takes user's current location and desired destination, returns Fare object w/ estimated fare and ETA
			- ```
			  POST /fare -> Fare (has price and ETA)
			  Body : {pickLocation, Destination}
			  ```
	* request ride:
		* confirm ride request after reviewing estimated fare:
			* ```
			  POST /rides -> Ride
			  Body {fareId}
			  ```
	- update driver locations:
		- ```
		  POST /drivers/location -> Success/Error
		  Body: {lat, long} #driverId in session cookie
		  ```
	* accept ride request endpoint:
		* ```
		  Patch /rides/:rideId -> Ride
		  Body: {accept/deny}
		  ```
* high level design:
	* input start location, destination: get estimated fare:
		* rider client: app for users
		* API gateway: route to microservices, do TLS termination, etc.
		* ride service: manages ride state, uses distance and travel time for fare estimates
		* use third party mapping API
		* database stores fares:
			* `fare: id, userId, source, destination, price, eta`
	* calculating a fare
		* rider enters pickup location and desired destination into client app, sending post request to backend through `fare`
		* forwarded to ride service
		* ride service uses third party mapping API to calculate distance and travel time
			* uses pricing model
		* ride service: creates new Fare entity in DB with details about fare
		* returns Fare entity to gateway, forwarding to Rider client
	* requesting a ride based on estimated fare:
		* add a ride table to database
			* has rideId, riderId, driverId, fareId, source, destination, status
		* flow:
			* send POSt to backend system w/ Fare ID
			* API gateway performs auth and forwards
			* ride service receives request, creates new entry and marks it as requested
			* assign a driver to the ride
	* match riders w/ driver nearby and available:
		* driver client: allows drivers to receive ride requests and do location updates
		* location service: managers location data of drivers, storing info in DB, providing ride match service w/ latest location data
		* ride matching service: handle incoming ride requests to match requests w/ best drivers (proximity, availability, driver rating)
		* flow:
			* user confirms ride request in client app (sends POST to backend w/ fareId)
			* ride service creates a ride object, forwards request to ride matching service
			* drivers are sending current location to location service, updating DB w/ latest at/long
			* matching workflow uses updated locations for closest available drivers
	* drivers accept/decline request and navigate to pickup/dropoff
		* add a notification service that gives drivers new ride requests
		* drivers can accept requests in a timely fashion that way
		* notifs sent through APNs (Apple Push Notification) or FCM (Firebase Cloud Messaging)
		* flow:
			* after ride matching service finds ranked list of drivers, sends notif to top driver
			* driver receives a notification that new request is available
				* they then accept request, sends PATCH to rideID
				* decline: go to next driver
			* ride service: updates to accepted, updates assigned driver, returns pick up locations
				* this endpoint: 
					*  ```
					  Patch /rides/:rideId -> Ride
					  Body: {accept/deny}
					  
					  ```
* deep-dives:
	* how to handle high frequency of writes:
		* ten million drivers send locations every 5 seconds
		* need to have query efficiency: avoid a full table scan
		* solutions:
			* batch processing: write a short number of updates aggregated over interval and processed
			* proximity searches: have a specialized geospatial database w/ appropriate indices
			* batch processing: reduces # of writes to a database, collects data over a predefined time interval and writes through batch op.
			* use a specialized index for quicker queries (geospatial databases use quad-tree to index driver location)
				* partition space into quadrants recursively
				* POSTGIS
			* issue:
				* interval b/w batch writes have a delay, position doesn't reflect most current (leads to suboptimal matches)
			* better:
				* use in-memory data store like redis
					* supports geospatial data types and commands
				* in-memory data store supporting geospatial data types through geohashing
				* geospatial commands like Geoadd and Geosearch
				* no need for batch processing since Redis can handle high volume of location updates
				* each Geoadd command overwrites previous loc. for driver
				* run a periodic cleanup that removes drivers whose last update timestamp exceeds threshold
					* maintain companion sorted set keyed by timestamp
					* periodically remove entries older than threshold
				* challenge:
					* durability: if system crashes or fails we lose data
					* add redis persistence (RDB) or AOF 
						* saves in-memory data to disk
						* redis sentinel: automatic failover if master node goes down, ensures replica is master
	* how to manage system overload from frequent driver updates while ensuring location accuracy
		* have an adaptive location update intervals
			* dynamically adjust frequency of location update based on speed, direction of travel proximity to pending ride requests (use client sensors)
			* if stationary don't update that often
			* moving a lot, update more frequently
		* issue:
			* need effective algorithms too determine the optimal update frequency
	* prevent multiple requests from being sent to the same driver
		* bad: application-level locking w/ manual timeout checks
			* each ride match service marks request as locked
			* can cause race conditions w/ a lack of coordination
			* if one instance sets lock but fails before releasing causes issue because no other instances have knowledge of the lock
			* scalability: coordinating locks is harder w/ more systems
		* better: database status update w/ timeout handling
			* move lock to DB
			* use build in transactional capabilities to ensure only one instance locks a ride
			* send a request, change status to "requested"
			* deny it, update status to available
			* simple timeout mechanism ensures that lock is released
			* issue w/ relying on in memory layout to handle unlocking if driver doesn't respond
				* service crashes/restarts, timeout lost and lock lost
				* so have cron job to check for locks and release them
		* best:
			* distributed lock w/ redis
			* ride sent to driver, lock created w/ unique ID and TTL of 10 seconds
			* lock acquired, no other service can send ride request to same driver until lock expires
			* driver accepts ride, Ride Matching Service updates ride status to accepted and lock is released + lock released
			* driver doesn't accept ride within TTL window, lock expires
			* issue:
				* reliant on in-memory data store (need monitoring/failover)
				* given locks only held for 10 seconds, reasonable tradeoff
	* avoid dropped requests:
		* with lots of requests, how to deal w/ the pressure
		* queue system w/ dynamic scaling
			* new requests are added to the queue
			* ride matching service processes requests in a FIFO way
			* queue too large: add more horizontal instances of ride-matching service
			* scale system dynamically on demand, no requests drop (can also parttion on geographic region)
		* can also use a distributed message system like Kafka
			* commit offset in the queue (so we know most recently read)
			* RIde Matching Service goes down, match request in queue and new instance picks up
		* issues:
			* queue needs to be scalable, fault-tolerant, and highly available
			* not basic FIFO: use a priority queue, prioritize requests based on factors like proximity, driver rating
	* what happens if ride request continues to be processed?
		* delay queue:
			* retry ride request w/ next available driver
			* schedule a delayed message in the queue in SQS
			* delayed message contains the ride request and the original driver
			* delayed message processed: system checks if ride unassigned
			* so flow:
				* system sends driver request
				* also puts in queue 
				* if driver accepts before 10 seconds, ride becomes assigned
				* delayeed message runs, check ride status
					* if already assigned do nothing
					* else send to driver B (next best driver)
			* issue:
				* complexity of multiple delayed messages and ensure they cancelled when driver accepts
				* if driver accepts ride after delay scheduled but before processed
					* ensure graceful handling
				* have consistency b/w delay queue and ride matching service
		* durable execution:
			* Temporal or AWS Step Functions
			* workflow is modeled as durable workflow w/ complex logic (driver timeouts, retries, fallback mechanisms)
				* automatically do mechanism
					* send ride request to first driver
					* set 10 second timeout
					* driver accepts complete workflow
					* declines or timeout, move to the next driver
					* continue process until driver found or everyone is exhausted
			* issue:
				* complexity of workflow orchestration (new concepts/tools)
				* but guaranteed execution, built-in fault tolerance
	* further reduce latency:
		* never do vertical scaling
			* theoretical limit, if the one machine goes we are fried
		* shard data geographically, use read replicas to improve read throughput
			* reduces latency and improves throughput
			* reduces latency by reducing distance b/w client and server
		* by sharding geographically, we increase throughput (more lines if that makes sense)
		* scatter gather across multiple shards is through proximity search on boundary
		* issue:
			* complexity of sharding and managing multiple servers
			* data distribution needs to be even
			* handle system failures and rebalance
				* consistent hashing, replication
				* scale horizontally while maintaining fault tolerance and high availability
* ids should be added to JWT and not passed in the body