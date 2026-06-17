- function requirements:
	- viewers post comments on a Live video feed
	- can see new comments posted while watching live video
	- comments made before joining live feed
- nonfunctional requirements:
	- system should scale to support millions of concurrent videos and thousands of comments per second
	- prioritize availability over consistency (eventual consistency is fine)
	- low latency (broadcast comments to viewers in real time)
- core entities:
	- video: video being broadcasted by a user
	- user: viewer or broadcaster
	- comment: message posted by user on a live video
- api system interface:
	- POST to create comment:
		- ```
		  POST /comments/:liveVideoId
		  Header : JWT | SessionID {
			  "message": "Cool video!"
		  } //user id isn't passed to request body, is instead a part of request header
		  ```
		* client: stores session token or JWT, passes to request header
		* server validates token, extracts userID
		* prevents tampering
	* fetch past comments:
		* ```
		  GET /comments/:liveVideoId?cursor={last_comment_id}&pageSize=10&sort=desc
		  ```
		* pagination important
* functional requirements:
	* post comments on live video feed:
		* users can just POST to the request
		* server validates request, stores comment in DB
		* ```
		  Comment:
			* has commentId, primary key
			* videoId, used to shard
			* author
			* content
			* createdAt (sort key)
		  ```
		* Client:
			* web or mobile app user can post comments on a live video feed using
				* authenticates user and sends to management service
			* comment management service:
				* responsible for creating/querying, receives comments from commenter client and stores them in db 
				* also gets comments and sends them to viewer client
			* database:
				* dynamoDB, no relational data or transactions
				* need fast scalable and highly available
		* flow:
			* draft comment
			* commenter client sends comment to comment management service
			* services: recieves request and stores comment
	* see new comments being posted while they watching live video
		* simplest approach: polling
		* client polls for new comments every few seconds
			* ```
			  GET /comments/:liveVideoId?since={last_comment_id}
			  ```
			* since is the last requeset id comment has seen
			* we return all comments posted after
				* client appends them to list of comments displayed
		* not feasible:
			* doesn't scale, # of comments, viewers grow, polling frequency needs to keep up
			* to get real-time comments, poll database every few milliseconds
	* see comments made before they joined live feed
		* need two things
			* immediately start new comments as posted in real-time
			* see history of comments posted before joined
		* scroll up to view progressively older comments
			* "infinite scrolling"
		* go get the N most recent comments occurring before a certain timestamp
			* have pagination (break. up large set of results into smaller ones)
		* two approaches:
			* offset pagination:
				* offset to specify starting point for fetching a set of results
				* offset 0 to load most recent, increases by number of comments as user scrolls 
				* example: `GET /comments/:liveVideoId?offset=0&pageSize=10`
				* issues:
					* inefficient as volume grows
					* database counts through all rows preceding offset for each query
					* not stable: comment added or deleting while the user is scrolling offset is incorrect
			* cursor pagination:
				* cursor to specify starting point for fetching results
				* cursor: unique ID pointing to specific item in list of results
				* cursor set to most recent, updated each time the user scrolls to load more
				* more efficient than offset pagination: DB doesn't count all rows preceding cursor
					* assuming index on cursor field
				* also stable: if deleted, cursor points to the right thing
					* ```
					  GET /comments/:liveVideoId?cursor={last_comment_id}&pageSize=10
					  ```
				* requires database query for each new page, can be significant in high traffic environment
* deep dives:
	* how to ensure comments broadcasted to viewers in real time?
		* push-based approach
		* server pushes new comments as soon as created
		* implement through Websockets or server-sent events
		* websockets:
			* two way communication channel b/w client and server
			* client opens connection w/ server and keeps it open
			* new comment arrives, server distributes to all clients, updates comment feed
			* issue:
				* useful for more balanced read/write ratios
				* comment creation is more infrequent than viewing
				* doesn't make sense to open a two-way communication channel for each viewer
				* mostly server writes
		* server-sent events:
			* persistent connection over standard HTTP
			* server can push data to client in real time, client to server communication is REST requsts
			* better solution for read/write ratio
				* infrequent comment creation uses POST requests
				* frequent reads benefit from one-way streaming
			* issues:
				* several infra challenges
				* proxies/load balancers lack streaming support
				* brows impose limits on concurrent sse per domain
		* new flow:
			* user posts comment, persisted to database
			* comment management service sends SSE to all connected clients
			* commenter client receives comment, adds to feed of all connected clients
			* client gets comment, adds to feed
	* how does system scale to support lots of users
		* support many millions of concurrent viewers, do on multiple servers
			* capacity of connections isn't 65,535 (every connection is identified by source IP, source port, destination IP, destination port)
			* so we can have hundreds of thousands to millions of connections 
		* user A watches live video 1 but connected to server 1
		* user B watches live video 1 but connected to server 2
		* how does comment to live video 1 propagate to all servers
		* naive pubsub:
			* every server processes every comment
			* messaging servers seperated (write < read)
			* round robin, connecting to real time messaging server
			* mapping in local memory for each server:
				* ```
				  "liveVideoId1": ["sseConnection1", "sseConnection2"]
				  ```
			* CMS publishes message to channel when new comment created
			* all realtime messaging servers subscribe, send comment to all videos
			* works:
				* inefficient thought because each server processes every comment even if none of connected viewers are watching
		* partitioned pub/sub
			* partition the stream into different channels
			* each server subscribes only to needed channels
			* can't have a channel per resource, distribute across N channels
			* use consistent hashing to ensure that viewers have affinity to particular servers (on liveVideoId) to route viewers of same video to same server
				* use NGINX or Envoy to ensure related viewers stay together
				* or store mapping of liveVideoId to server in coordination service like Zookeeper
			* challenges:
				* complexity of managing load balancing and pubsub subscriptions
				* server viewer comp changes, update subscriptions 
					* servers scale up or down, routing mappings need to stay in sync
		* dispatcher service:
			* no pubsub
			* route each new comment to correct Realtime Messaging Server
			* dispatcher: dynamically maps server to video (sync w/ heartbeats and registration protocols as servers join or leave)
				* depending on message routes
			* eliminates need to manage pub/sub subscriptions, centralize routing logic and add more sophisticated routing 
			* multiple dispatchers can be managed w/ load balancer
			* issue:
				* need to have refreshed view of system frequently
		* different tradeoffs in pub/sub systems
			* kafka: scalable, fault tolerant
				* struggles with changes
			* redis pub/sub: low latency, handles dynamic susbcriptions
	* mega streams: 
		* lots of comments and and lots of viewers come in 
		* different problem: not meaningful to show most recent because of how quick it is
		* solution:
			* show a representative subset
				* need to see enough comments
				* sample comment stream and only deliver a subset to each viewer
				* sampling rate based on comment velocity
					* 100 per second, sample 50%
				* clever sampling strat:
					* prioritize comments from users from viewer follows, comments getting reactions, comments from verified accounts
				* challenges:
					* persistent SSE connections to millions of concurrent viewers
			* CDN-based delivery w/ periodic snapshots
				* pull-based on CDN infra
				* server side: maintain ring buffer of most recent 100-220 comments for mega stream
				* every second take a snapshot and write to redis or CDN
					* CDN distributes snapshot globally, caching at edge locations
				* client side: polll CDN every second for latest snapshot
					* client doesn't dum all comments but queues them over polling interval for simulating a stream
				* leverages existing infra (CDNs serve same content to millions of users)
					* have a dynamic threshold (stream crosses particular threshold or comments flip from SSE to CDN)
				* tradeoff:
					* latency:
						* delivering comments w/ 1-2 seconds of delay but doesn't matter since mega stream
						* need to insert own comment
						* transition back to SSE when viewership drops
	* disconnection:
		* can do nothing (reconnect, start receiving new comments from that moment)
			* any comments during post is lost
		* issue:
			* poor user experience
			* miss tense/important meetings
		* great solution:
			* last even id
			* SSE has built-in mechanism for handling reconnection w/ `Last-Event-ID`
			* every comment: includes unique event ID
			* browser detects disconnection, automatically reconnects and includes last event ID received
			* server side: replay missed comments before normal streaming
			* client can also track last comment ID locally, catch up w/ HTTP
				* `GET /comments/:liveVideoId?since={last_comment_id}&limit=100.`
			* mobile: smarter about battery life
				* preemptively disconnect SSE and record position
				* when foregrounded, reconnect and catch up
				* more efficient than maintaining connection while backgrounded 
				* be smart about how far we replay:
					* user disconnected for an hour, thousands of stale comments dosn't make sense
					* replaying five minutes is reasonable, degradation for longer gaps
				* challenges:
					* user connects to diff realtime messaging server, server needs access to comment history
						* shared Redis cache (replay recent comments)
					* SSE stream and HTTP catch-up need to coordinate to avoid duplicates
						* SSE comments arrive while processing HTTP request: deduplicate and merge
