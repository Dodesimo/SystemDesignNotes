- functional requirements:
	- create posts featuring photos, videos, caption
	- follow other users
	- see a chronological feed of posts from users they follow
- non functional requirements:
	- system should be highly available (availability of photos/videos over consistency)
	-  deliver feed content with low latency
	- render photos and videos instantly (low latency media discovery)
	- scalable to support 500M DAU
- core entities:
	- user: stores information like username, profile detials
	- post: reference to media file, caption, and user who created it
	- media: actual bytes of the media file (S3)
	- follow: represents relationship between two users (one user follows another)
- API system interface:
	- create a post:
		- ```
		  POST /posts -> postId {
			  "media": photo/video bytes
			  "caption": ""
		  }
		  ```
	* follow others:
		* ```
		  {
		  POST /follows {
			  "followeeId": "123"
		  }
		  }
		  ```
			* the follower's ID is taken from the auth token
	* see their feed:
		* ```
		  GET /feed?cursor={cursor}&limit={page_size} -> Post[]
		  ```
* high level design:
	* making a POST:
		* client makes a POST request to API gateway w/ media and caption
		* Gateway routes to the Post Service
		* Post Service receives media + caption, stores post metadata in database, actual bytes in a blob store
		* Post Service returns a `postId` back to the client
	* following others users
		* maintain a Follower's table storing followerId and followedId
		* insert a new row to table every single time
		* dedicated Follow Service to handle follow/unfollow operations seperately from Post Service because these two have different behaviors (scale and optimize each separately)
	* chronological feed of posts:
		* get followees: query the `Follow` table and get a list of `user_ids` that the current user follows
		* query the `Post` table to get recent posts for each followed users
		* Merge and sort by combining the retrieved posts and sort them chronologically through timestamp
		* return the sorted posts to the client
		* add indices to the database to avoid full table scans
		*  to make queries faster:
			* add indices to database to avoid full table scans
				* follower table: partition key of followerId, sort key of followedId
				* post table: partition key of userId (since most of our queries try to get the posts of a given user)
					* sort key: composite of `createdAt` and `postId`
* potential deep dives:
	* delivering feed content w/ low latency
		* fan out on read doesn't work and scale
			* for users following 1,000+ accounts we need 1,000+ queries
			* lots of read amplification: every time user refreshes feed, generates lots of reads
			* repeated work: if two users follow many of the same, querying same posts (popular posts could be retrieved millions of times)
			* unpredictable performance: latency depends on the number of accounts user follows
		*  this is because we postpone computation until read time which is when users want results
		* solutions:
			* bad solution: simple caching (improvement on fan-out on read)
				* add a cache in front of the Posts table through Redis
				* posts are in this cache (if not present, query the database and store the results for future requests)
				* key the cache w/ a `user_id` and a `cursor` (allows a specific page of the cache to be fetched, returns a list of postIds)
				* issue: still not fast enough
			* good solution: precompute feeds (fan-out on write)
				* instead of generating feed when user requests, generate it when a user posts
				* user creates post, query Follows table to get all users who follow posting user
				* GSI on Follows table w/ followedId as the partition key and the followerId key 
					* find all followers of a user who just posted
				* for each follower: prepend new postId to the precomputed feed
				* precomputed feed can be stored in a dedicated table or a cache
					* store it as a sorted set in Redis
					* members are postIds and scores are timestamps
					* when user wants. posts: need Top N posts which is a single fast operation
					* requesting feed requires hydrating posts based on `postIds`
						* for each id in the cache, get the metadata from the Posts table in DynamoDB (but this introduces latency)
						* cache entire post metadata in Redis (requires more memory and introduces data consistency challenges)
						* hybrid: two Redis structures, one for feed postIds and another for a short-lived post metadata cache
								* so we get all the metadata from the Redis post cache
								* for any misses, fetch from DynamoDb
								* update the post cache w/ the fetched metadata
								* return combined results
					* post gets created:
						* store it in the table and then media in blob store
						* put the postId on a queue that gets asynchronously processed by fan-out service
						* fan-out service then queries the Follows table to get all users that follow the posting user
						* add the postId to precomputed Redis feed
						* when user requests feed:
							* read top N from sorted set
							* hydrate based on post Ids (see if in the Redis post cache)
							* combine results and then return to the user
					* issue w this: single post by a user triggers millions of writes to Redis
			* great hybrid approach:
				* celebrities: have fanout on read 
				* for users: we do fanout on write
				* define a threshold for the # of followers, if its less than that threshold, precompute feeds but for more than, don't precompute
				* so when celebrity posts:
					* posts table gets added to, don't trigger an asynchronous feed update for followers
					* user requests feed: fetch precomputed portion from normal users, then query the posts for celebrities and then merge the two
					* effective mix between pre-computation and real-time merging
				* need to carefully tune the 100,000 follower threshold + manage complexity + avoid creating inconsistent user experience since users following many celebrities experiencing slower feeds
				* for redis: implement Redis Sentinel for high availability and use Redis Cluster for data sharding across multiple nodes and persistence such as Append-Only File or RDB snapshots
	* render photos and videos instantly:
		* large media files: worry about upload efficiency and downloading latency
		* use AWS S3's multipart upload API
			* get a pre-signed URL to upload the media (URL is valid for a limited time) and allows user to upload directly to S3 without having to go through servers
			* Client Side: multipart upload API to upload file in chunks to the pre-signed URL
			* S3 reassembles chunks and stores the final file in S3
			* metadata has a upload status field set to `pending`, when the media is uploaded the post is updated to complete
				* two ways to handle the change
				* client-driven: PATCH request to update the post metadata with the S3 object_key and set the upload status to complete
				* server-driven: configure S3 event notifications to trigger a Lambda function or background job when upload completes
					* this is better since we don't have to rely on the client to be reporting the correct information
		* use CDN for viewing
			* don't directly download data from the S3 bucket
			* CDN: configure w/ a TTL so that we reduce latency for users and load on our origin S3 bucket
				*  configure to cache based on type/popularity (images get cached for 24 hours at edge locations)
			* but we force all devices to download/access the same media at the same resolution
			*  media is uploaded, use a media processing service to generate multiple variants optimized for different devices/network conditions (dynamic media optimization)
					*  for images: different resolutions and formats
					* for videos: multiple quality levels, adaptive streaming
			* CDN then serves the optimized variants based on the requesting device/network conditions
			* intelligent caching strategies such as popular media is cached aggressively at edge locations while less accessed content has shorter TLLs
		* scaling:
			* move old data to cheaper storage like Glacier or DynamoDB data to S3
			* horizontally scale microservices to handle load
			* 