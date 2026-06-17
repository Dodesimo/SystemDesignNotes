- only allows set # of requests before giving a HTTP 429
- core requirements:
	- identify clients w/ user ID, IP address, or API key
	- limit HTTP on configurable rules (like 100 API requests)
	- limits exceed, reject w/ HTTP 429
- non functional requirements:
	- 1 million requests per second across 10 million requests per second across 100 million daily users
	- minimal latency overhead (< 10 ms per request check)
	- highly available (eventually consistency is OK)
- core entities:
	- rules:
		- rate limiting policies defining limits for different situations (specifies params, what client it applies to, what endpoints covered)
			- authenticated users get 1000 requests/hour
	- clients:
		- entities getting rate limited (could be users, IP addresses, API keys, combinations)
			- has a rule limiting state tracking current usage against applicable rules
	- requests:
		- requests that need to be evaluated against rate limiting rules
			- has info like client identity, endpoint accessed, timestamp determining what rules apply and track usage
	- flow:
		- request arrives, identify client, apply rules, and check usage
- system interface/design:
	- rate limiter: other services call to check so this is an infra component
	- ```
	  isRequestedAllowed(clientId, ruleId): {passes: boolean, remaining: number, resetTime : time}
	  ```
		* returns information about response headers
* high level design:
	* where should we place the rate limiter?
		* bad: in-process:
			* each server/microservice has direct rate limiting
			* when request comes in, server checks in-memory counters, updates, and decides to allow/reject request
			* issue:
				* not global, each server only knows about its own traffic,
				* underestimates counts (5 servers, each server sees 20 requests per minute, so falsely thinks we are well under the limit)
				* if load balancer changes how traffic gets routed, limits are unpredictable
		* good: dedicated service:
			* rate limiter is a microservice between clients and application servers
			* client first makes API call to rate limiting service, checks counters, and responds w/ allow or reject w/ 429
			* allows for more flexibility, application servers can provide far more rich context when doing rate limit checks
			* guarantees you global state
			* issue:
				* adds latency to every request (additional network round trip before processing)
				* added an additional point of failure
					* rate limiting service goes down: how do we process requests?
					* consider how to handle network issues gracefully
						* what if rate limiter is slow to respond
		* best:
			* API gateway/load balacner
			* runs on edge, integrated into API gateway or load balancer
			* examine request, check IP address and user authentication headers and API keys
			* applies limits, forwards request downstream and returns 429
			* the servers only deal with valid, legitimate traffic
			* challenges:
				* limitation is context, only has access to header information in the HTTP request
					* can't implement easy business logic
			* where to store rate limiting state
				* need quick access to counters/timestamps, add an in-memory store
	* identifying clients:
		* have access to information in HTTP request
		* don't want to make external calls to databases or other services due to latency
		* unique aspects:
			* User ID: each logged-in user gets their own rate limit allocation (in the authorization header as JWT token)
			* IP address: good for public APIs, but doesn't work behind NATS or corporate firewalls (present in the `X-Forwarded-For`)
			* API key: every holder gets own limits, denoted in X-API-Key
		* have combinations of restrictions
			* per-user limits: 1000 requests/hour
			* per-IP limits: 100 requests/minute per IP
			* global limits: 50,000 request/second total
			* endpoint-specific limits: search API is only particular amount, but other endpoints are different
			* follow the most restricted limit
	* what rule determines restriction of requests:
		* fixed window counter:
			* divide time into fixed windows and count requests in each window
			* for each user, maintain a counter that resets to 0 at the start of each new window
			* implementation: hash table mapping
				* ```
				  {
					  "name:time": counter
				  }
				  ```
				* can request 100 requests at edges and get a lot of requests in short period of time
		* sliding window log:
			* new request arrives, remove all timestamps older than window
			* see if remaining count exceeds limit
			* issues:
				* lots of memory usage (user making 1000 requests per minute, store 1000 time stamps)
				* doesn't scale to millions of users (run into memory problems)
		* sliding window counter:
			* approximates sliding windows w/ fixed windows
			* counter for current window and previous window,
				* request arrives, estimate how many requests user should have made
				* done by weighting previous and current windows
			* gives more accuracy than fixed windows while using minimal memory
				* assumes that traffic distributed within windows
		* token bucket:
			* client has a bucket holding certain number of tokens (burst capacity)
			* tokens added at a steady rate
			* each request consumes a token
			* no tokens available, reject request
			* simple to implement, track tokens, last_refill_time per client
				* need to choose the right bucket size and refill rate (handle cold starts)
		* each bucket tracks current token count and last refill timestamp
		* we need a centralized state (otherwise each local server thinks its the global state)
			* use Redis: fast, in-memory data store all gateway instances can access
		* token bucket algorithm:
			* request arrives w/ a user ID
			* gateway calls Redis to fetch current bucket state through HMGET user:bucket tokens last_refill
			* based on last_refill, add tokens to the bucket
				* alice's bucket last updated 30 seconds ago, refill rate is 1 token per second, gateway adds 30 tokens
			* gateway updates bucket state atomically through a Redis transaction to prevent race conditions
				* ```
				  MULTI
				  HSET user:bucket tokens <new token>
				  HSET user:bucket last_refill <current>
				  EXPIRE alice:bucket 3600
				  EXEC
				  ```
				* ensures all commands are in a atomic operation
			* then gateway updates token count (1 token left, allow request, no tokens left reject request)
				* race condition: because read operation happens outside transaction, so if two requests appear at the same time, both could update the bucket at the same time
			* solution:
				* move read-calculate-update logic into an atomic operation through Lua scripting
					* entire rate limiting decision is atomic
					* scrip: reads current state, calculates token count, updates bucket in one go
			* why do we want redis?
				* super fast
				* cleans up through EXPIRE command
				* very highly available because multiple instance replication
				* atomic operations ensure 0 race conditions
	* limits exceed, reject requests w/ HTTP 429
		* do requests get dropped or queued?
			* most just drop them
			* why not queue?
				* consume memory and processing resources
				* think requests failed and retry causing more server load
				* queue can cause delays that are unpredictable
		* how to make response helpful?
			* 429 "Too Many Requests" along w/ headers that help clients understand what happened and how to recover
				* X-RateLimit-Limit: request ceiling
				* X-RateLimit-Remaining: # of requests left
				* X-RateLimit_Reset: when it resets
* Deep dives:
	* how to scale to 1M requests/second
		* typical redis instance handles 100,000 to 200,000 ops/second 
		* need to partition redis load across multiple instance 
		* need to shard consistency so that all client requests end up at the same redis instance
			* this allows for rate limiting state to always end up in the same shard
		* request arrives, extracts ID and applies distribution algorithm, routes rate limit check to correct Redis instance
			* Redis Cluster: better than managing individual Redis instances yourself
			* automatically handles data sharding by dividing keys across numerous hash slots
	* how to ensure high availability and fault tolerance
		* how to deal w/ failiure mode?
			* rate limiter can't reach Redis, reject all requests w/ 503 service unavailable or HTTP 429 responses (fail close)
			* issue:
				* API goes offline during Redis outages
				* failed requests even if backend services are healthy
				* better in situations where we don't want to process for security
					* Financial systems processing payments: reject transactions
					* High security environments where uncontrolled access in worse
			* allow all requests to proceed without rate limiting (fail open)
				* skips rate limit checks, forwards to backend (keeps API available)
				* issue:
					* we lose temporary rate limit protection (backend services could get overwhelmed)
					* dangerous for social media platforms (viral events stressing situation can cause a total failure)
		* best option: depends on platform
		* standard solution for Redis availability: master-replica replication
			* each shard gets one or more replicas synchronizing w/ master
			* when a master fails, replicas get promoted to new master
				* works well w/ Redis Cluster (has built-in failover capabilities that detect master failures and promote replicas without manual intervention)
			* issue:
				* costs more, requires handing replication lag
		* need to have monitoring/alerting for high availability
			* maintain health metrics like CPU usage, memory consumption and network connectivity for Redis
			* need application-level monitoring of rate limiting success rates, response latencies and alerts when we enter fail-open
	* how to minimize latency overhead?
		* each check is a network round trip to Redis, adding latency to user requests
			* Redis operations are sub-millisecond, overhead can add several milliseconds per request
		* connection pools: 
			* each rate limit check doesn't cause a new TCP connection
			* gateways maintain a pool of persistent connections
				* eliminates TCP handshake overhead 
				* allows connections to be reused across multiple requests
			* most Redis clients handle connection pooling automatically, but need to tune pool size
		* geographic distribution provides most latency reductions:
			* deploy infra close to users through API gateways and Redis clusters in multiple regions
				* can accept eventual consistency between regions in exchange for lower latency
		* can also cache rate limit state (risky if stale cache data leads to incorrect decisions)
			* can also batch operations 
			* but these things add complexity (just maintain closeness and have connection pools)
	* how to handle hot key situation:
		* single user or IP generates enough requests to overwhelm a shard
		* could be abuse or high-volume clients
		* solution:
			* legitimate high-volume:
				* client side rate limiting: well-behaved clients have their own rate limiting to smooth patterns
					* prevents users from creating hot shards while reducing load
				* client can batch multiple operations into a single request, reducing total number of rate limit checks
				* premium tier: offer more infra/limits for users who need them
			* abusive traffic:
				* temporarily block IP/API w/ block list
					* kept in a shard, checked in the case of Cache misses
				* DDoS protection: use Cloudflare to block traffic
		* can have situations where legitimate users share IP addresses
			* design rate limits to account for this (more for IP-based)
	* dynamic rule config:
		* need to have flexibility to adjust limits without deploying new code
		* poll:
			* database or dedicated config service stores changes
			* API gateways poll for config changes and update logic, and cache
			* challenges:
				* update delay (window between rule change and how it impacts all gateways)
				* emergency-reduction can be problematic for urgent situations (need to wait a bit for changes to propagate)
		* push:
			* config changes are immediately sent to all gateways
				* what Zookeeper designed for (distributed config. w/ real time notifs)
				* maintains config data and notifies all clients when config changes
			* other options:
				* use redis pub/sub that maintains persistent connections to gateways
			* operator changes rule limit, config service notifies all connected gateways that update rules
				* issues:
					* adds complexity
					* handle connection failures, ensure gateways get updates, deal w/ partial failures
					* need fallback mechanisms when push is unavailable