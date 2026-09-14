- core requirements:
	- functional
		- post item for auction w/ starting price and date
		- bid on item only accept bids if they are higher than the current
		- view an acution current highest bid
	- non functional:
		- strong consistency for bids to ensure all users see same highest bid
		- fault tolerant and durable
		- display current highest bid in real time
		- support a lot of concurrent auctions
- core entities:
	- auction: info like starting price, end date, and item
	- item: name, description image
	- bid: amount bid, user, auction
- API:
	- POST, takes auction details and returns created auction
	- ```
	  POST /auctions -> Auction and Item
	  {
		item: Item
		startDate: Date,
		endDate: Date,
		startingPrice: number,
	  }
	  ```
	* placing a bid: POST endpoint that takes bid details and returns created Bid
	* placing Bids: POST takes bid details and returns created Bid:
		* ```
		  POST /auctions/:auctionId/Bids -> Bid
		  {
			  Bid
		  }
		  ```
	* get auction details:
		* ```
		  GET /auctions/:auctionId -> Auction & Item
		  ```
* high level design:
	* post item for auction w/ starting price and end date:
		* add auction service, connects to database, stores auction/item data after POSTing to `/auctions` endpoint
	* bidding:
		* bidding service:
			* validates incoming bids (check the bid amount is higher than current max)
			* update the auction w/ highest bid
			* store bid history
			* notify relevant parties of bid updates
		* why is bidding different than auction?
			* independent scaling: since bidding traffic is more than auction traffic
			* isolation of concerns: bidding has more complex logic around validation/race conditions/real-time updates, maintain clean service boundaries
			* isolate high-write bidding service for optimization versus auction high-read service
		* flow:
			* client posts to `/auctions/:auctionId/bids` with details
			* request to the bidding service
			* bidding service queries database for highest current bid (if the bid in DB is less than new one, accepted, else rejected)
			* bids stored in database in new bids table
	* view an auction:
		* GET request: `/auctions/:auctionId`
		* refresh the maximum bid price: don't want to bid based on a stale amount, poll for latest maximum bid price every few seconds
* potential deep dives:
	* strong consistency:
		* what if two people simultaneously update the bid what happens
		* bad solution:
			* row locking w/ bids query:
				* row level locking, on the bids table on
				* begin transaction w/ database transaction
				* lock all bid rows for auction
				* query maximum bid from locked rows
				* compare new bid against it
				* write the new bid if accepted
				* commit the transactions
				* challenges:
					* doesn't prevent concurrent inserts
					* both read the same auction and insert the same thing
		* good: cache the max bid externally in some type of redis
			* read cache for an auction from redis
			* update the cache: if new bid higher, update cache w/ new max bid
			* write the bid to db w accepts/rejects
			* issues:
				* read/write of cache needs to be atomic to avoid a race condition
				* redis: single-threaded so supports atomic operations
				* cache needs to be strongly consistent w/ the database
					* so either accept redis as the source of truth and write to DB async
					* optimistic concurrency and retry: update cache atomically, db write fails, roll back
					* write to db first, update the cache
						* if cache updates, invalidate cache entry and repopulate next read
			* better: cache within the db itself
				* we add a max field to the auction table
				* read the max bid for the auction from the auction
				* write the bid to the DB w/ accept or reject status
				* update max bid in auction table if new bid is higher
				* use OCC (its like CAS): only update auction to new max bid if greater and if previous max bid hasn't changed
		* how to ensure system is fault tolerant and durable?
			* add a durable message queue that gets bids on it
			* why?
				* if bid comes in, write to queue immediately
				* even if the service fails, messages stored in the bid
				* buffer against load spikes (lots of bids per second): without message queue we would drop bids, crash under load, over-provision servers (expensive)
				* allows for ordering to be enforced per auction: partitioning by auctionId, all bids for the same auction land on the same partition, procesed in the order thei received
					* important for fairness
			* use kafka:
				* high throughput: handle millions of messages
				* durability: messages are persisted to disk, replicated, we don't lose a bid
				* partitioning: queue gets partitioned by auction id, so everything is processed in order while doing parallel processing of bids
			* flow:
				* user submits bid
				* API gateway routes to a producer, writes the bid to kafka
				* Kafka acknowledges the write, tell the user bid received
				* bid service consumes at a rate from the topic
				* bid valid, write to DB
				* bid fails, retry
		* how to display highest current bid in real time:
			* can't just poll: too slow (every few seconds not quick enough for an auction)
			* inefficient: client hits DB every request (wasteful since max bid hasn't changed)
			* solutions:
				* long polling: maintain an open connection till new data or timeout
				* client establishes connection, remains open till new data arrives or timeout happens
				* server holds request open, doesn't response immediately
				* new bid accepted, server responds to all waiting request w/ updated max bid
				* client then immediately initiates new long-polling request
				* within client, keep trying requests to max-bid and on server side, maintain map of pending requests, when new bid accepted, respond to all relevant waiting requests w/ new bid
			* issues:
				* long polling has limitations
				* server needs to maintain open connections for all clients which can consume significant resources when dealing w/ popular auctions
				* also can cause a "thundering herd": all clients connect simultaneously after an update
				* timeout causes delay: bid comes in just after client's long poll times out, won't see update till next request
					* tradeoff b/w resource usage and latency
				* scaling is hard b/c every server maintains own set of connections  (if scale horizontally, need to redistribute requests)
			* best:
				* server sent events: 
					* unidirectional channel from server to client, server can push updates whenever without client polling
					* user views auction: browser establishes sse connection
						* push new bid values through this connection whenever they change
					* client adds the bid-stream as an event source, on an event it updates the UI w/ it
					* server: maintains a map of string to responses
						* id to to set of connection objects
						* for new bid, get the connections and then for each connection send request
					* issue:
						* when user base grows, need multiple servers to handle SSE connections
						* creates problem w/ coordination: new bid comes in to server A, users watching that same auction connected to server B
						* server A doesn't know what what users are like this
							* can only know about its own connections
						* need a centralized place
		* support 10M concurrent auctions:
			* 10M concurrent auctions, every auction has about a 100 bids over life time, average auction runs for a week, we get 1,400 bids per second
			* design for roughly 10x (15,000 bids per second)
			* message queue: partitioning allows for parallelism (different instances can process bids for different auctions concurrently)
			* bid service: horizontally scale this because stateless service
				* auto-scaling, ensure running the right number of servers to handle current load based on memory
			* database: 
				* each auction is 1kb, bid is 500 bytes, average auction runs for a week, 10M * 52 = 520 million auctions
				* then 520M * (1kb + (0.5 kb * 100 bids)) = 25 TB
				* decent amount, but nothing crazy since modern SSDs can handle 100+TBs
				* shard tho
				* write throughput: 15K writes per second is more than what a Postgres instance handles
					* shard the database by auction id so that write load gets spread across multiple instances
			* SSE:
				* use pub sub where Bid Services are subscribed 
				* Server A receives a bid, then all other instances are notified of this bid through the pub/sub and get the message
					* if bid is for auction that client connected to, send that connection to new data
			* deep dives:
				* dynamic auction end times:
					* maintain auction end time in auction table w/ each new bid
					* cron job: check if it expires
					* more stuff:
						* schedule a task after an hour after a bid arrives
						* when task fires see if bid is still latest, if yes end auction
				* purchasing:
					* send email to winner asking them to pay, if they fail to pay, go to next
				* bid history:
					* have a list of bids throughout