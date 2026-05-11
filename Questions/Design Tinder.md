Core functional requirements:
- Users create a profile with preferences, specify maximum distance.
- Users can view a stack of potential matches in line with preferences, max distance of current location.
- Swipes (yes or no)
- Users get a match if they mutually swipe.
Non-functional requirements:
- Strong consistency: user sipes yes, then we should get a match notification
- Scale to a lot of daily users/concurrent users.
- System should load potential matches with low latency.
- Avoid showing user profiles swiped on.
Core entities:
- Users: person using app and profile that might be shown to the user.
- Matches: connection between 2 users from swiping "yes"
- Swipe: "yes" or "no" on a user profile, belongs to a user (swiping user) and another user (target_user)
API:
1. For creating a profile:
	1. ```
	   POST /profile {
		   min_age: 
		   max_age:
		   distance:
		   "interestedIn"
	   }
	   ```
2. For fetching a feed (don't pass in age, interests, etc. because we can load them server-side)
	1. ```
	   GET /feed?lat={}&long={}&distance={} -> User[]
	   ```
3. Swiping:
	1. ```
	   POST /swipe/{userId} (have all user information passed in headers for security) {
		   decision: 
	   }
	   ```
High-Level Design:
- Create a profile and specify a maximum distance.
	- Take request `POST /profile`, persist settings in a database.
	- Client -> API Gateway/Load Balancer -> Profile Service -> Profile DB
		- Profile DB: store info. about user profiles, preferences, other information.
	- Create a profile: 
		- POST to the `/profile`, route this request to Profile Service, update the user's profile preferences in the database, results returned to client through API gateway.
- View a stack of potential matches:
	- Easiest thing:
		- Query database for a list of users matching user's preferences and return them to client.
		- Account for location.
		- Example query:
			- ```sql
			  SELECT * FROM users
			  WHERE age BETWEEN 18 and 35
			  AND interestedin = 'female'
			  AND lat BETWEEN userLat - maxDistance AND userLat + maxDistance
			  same thing would apply to LONG
			  ``` 
		* When user request a new set of profiles:
			* Client: sends GET request to `/feed`
			* Gets routed to Profile Service
			* Profile service will query User Database.
			* Results returned to client w/ API Gateway
* Swipe on profiles:
	* Swipe service: persist swipe, check for matches
	* Swipe database: store swipe data for users.
		* We separate the DB here because writes are more frequent for swipes versus profile creation.
		* Allow us to scale the two services independently.
	* Lots of data: so we would have to partition.
	* Use a write heavy database like Cassandra (partition by `swiping_user_id`).
		* Allows us to easily see if a user swiped on another.
		* Write optimized through CommitLog + Memtables + SSTables
		* But we have eventual consistency.
	* Flow:
		* Client sents a post request w/ swipe direction.
		* API Gateway routes this request to Swipe Service
		* Swipe Service updates Swipe Database w/ the swipe data.
		* Swipe Service checks if there's an inverse, if so return a match.
* Get a match notification:
	* Immediate for the person who swiped second.
	* For the person who swiped first, send a push notification.
	* Flow:
		* Person A swiped right on Person B and we persisted this.
		* Person B swipes on Person A.
		* Give Person B a notification.
		* Person A gets a push notif.
Deep Dives:
- How do we ensure swiping is consistent and low-latency?
	- What if two people swipe at the same time?
		- Could argue that consistency isn't even necessary for this problem and prioritize availability.
	- Solutions:
		- Polling: this is dumb because it doesn't check immediately.
			- Also increases latency between polls meaning we don't get immediate feedback.
		- Transactions:
			- Swipe and check for reciprocal swipe happen in the same transaction.
			- Not as powerful as true ACID transactions.
				- Doesn't support multi-partition atomicity or isolation or rollback.
				- Also a lot of complexity: multiple round trips to achieve consensus.
			- Can't store all data in a single partition.
		- Shared Cassandra w/ Single Partition Transactions.
			- Create table that has a compound primary key ensuring swipes b/w users are in the same partition.
			- User swipes: sort the keys so that its in the same partition.
			- Cassandra single partition transactions are atomic.
				- Ensure that swipes b/w two users are in the same partition.
				- This allows for atomic checks for matches.
			- Issues: 
				- We would have to clean up our data to prevent partitions from infinitely growing.
		- Redis:
			- Guarantees consistency and atomicity, while Cassandra is for durability.
			- Create a compound key of the user.
				- So this means that both users have their swipe stored in the same partition.
				- Then we search for the other swipe in the same partition.
				- if both are matches, then return match.
			- Challenges:
				- Need to handle node failiures and rebalancing consistent hashing ring.
				- Flush swipe data to Cassandra and only have recent swipes in redis.
				- Hybrid approach: Redis is strong consistency and atomic, Cassandra has high durability and storage capabilities for historical data.
			- Main idea: Redis serializes to avoid a race condition that exists in a cross-partition check by having swipes for both users land on the same partition.
- Low latency for feed generation:
	- Solutions:
		- Index popular fields for feed generation (such as user preferences, age range, and geospatial data such as location).
		- Can also use a search-optimized database like Elastic Search and Open Search.
		- How to maintain data consistency between a primary transactional database and an indexed search database?
			- Change data capture mechanisms.
			- Changes in the database are asynchronously processed into Elastic Search.
			- Batch since Elastic Search is for read-heavy workloads.
		- Precomputation/Caching:
			- Have a background job generate feeds based on user preferences and locations, storing them in cache.
			- Schedule them during off-time hours.
			- Challenges:
				- Active users may exhaust their cached feeds.
				- User goes through pre-computed cached stack (expensive query).
		- Mixture of precomputation/caching:
			- Pre-compute periodically for users.
			- Users swipe and exhaust: system transitions to generating matches in real-time (Elastic search queries).
			- Trigger refresh few swipes left.
		- Could have stale feeds if preferences or locations change.
		- Have a strict TTL for cached feeds, recompute via background job on a schedule.
		- Precompute feeds for truly active users vs all users.
			- Bunch of tunable parameters: TTL for cached profiles, # of profiles cached, set of users we're caching for.
		- For user actions like changes to location/preference, we trigger a refresh in the background.
- Avoid showing user profiles they already swiped on:
	- Bad solution:
		- Contains check and a db query: check the partition for swipes.
		- issue: system that prioritizes availability over consistency, not all swipe data might have been replicas of a partition. 
			- Also, if user has a long swipe history, lots of IDs returned, so contains check gets more and more expensive.
	- Better: have a client-side cache and then maintain the same check and DB query.
		- Managing a cache to store data before data is fully consistent is expensive.
		- Client is a part of the system, have the client store recent swipe data.
		- Cache also useful when we almost exhausted cache:
			- Almost at the end, client pings backend to generate a new feed.
			- Request the feed.
			- Filter out profiles the user had eventually swiped on.
		- Challenges:
			- users with more extensive swipe histories and large userID contains checks that get slower.
	- Better:
		- Bloom filter: swipe history exceeds certain size, causes slow checks, build + cache filter.
		- Yield false positives: assume a user swiped on a profile.
		- But never generate false negatives.
			- Can be tuned in terms of false positive rates/
		- Challenge:
			- Bloom filter cache: easy to recreate w/ swipe data but expensive at scale.
		- 
	