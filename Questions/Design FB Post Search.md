- functional:
	- create and like posts
	- search posts by keyword
	- get search results sorted by recency or likecount
- nonfunctional:
	- system fast, median queries should return in < 500ms
	- support high volume of requests
	- searchable in a short amount of time, < 1 minute
	- all posts discoverable, old or unpopular posts (acceptable to have more time)
	- highly available
- scale estimation:
	- how much data stored?
	- how often we writing?
	- how frequency we reading?
	- math:
		- writes
			- posts created: `1B users * 1 (post/day) / 100k (seconds/day) = 10k posts/second`
			- likes created: `1B * 10 likes/day / (100k seconds/day) = 100k likes/second`
			- there is more likes than # of posts created
		- reads:
			- searches: `1B users * 1 (search/day) / 100k (seconds/day) = 10k searches/second`
		- storage requirements: 
			- `Posts searchable: 1B posts/day * 365 days/year * 10 years = 3.6T posts`
			- `Raw size: 3.6T posts * 1kb/post = 3.6 PB`
			- this is assuming metadata is a kilobyte
			- we have 10 years of history
- core entities:
	- user: creates posts
	- post: has a searching `content`, created by a `user`, implicitly has a like count
	- like: created when user likes post, care about count of likes
- API system/interface:
	- two: query path for searching, writing path for creating posts/likes
	- writes:
		- ```
		  //create a post
		  POST /post
		  //like a post
		  POST /post/{id}/like
		  ```
	* reads:
		* ```
		  //search
		  GET /search?query=THE_QUERY&sort_order=LIKES|TIME
		  ```
* high level design:
	* create a post:
		* requirement: write path, users to create and like posts through above calls
		* there's a  post service and a like service that creates Posts and Likes, which then get ingested into the index (containing content and likes)
	* search a post by keyword:
		* client searches api gateway, access search service, access Index (contains post w/ content, likes)
		* index has to search a lot of data for posts that contain keyword, better scaling options
			* horrible: 
				* key all posts in RDB
				* query w/ `SELECT * From posts LIKE '%$keyword%'`
				* issue: 
					* very slow, DB looks at every post and sees if content has keyword at query time
			* great:
				* created an Inverted Index
				* create dictionary mapping keywords to documents containing them
				* ```
				  keywords -> value is array of post ids
				  ```
				* search: grab key and then return associated posts
				* use Redis: keep indices in memory, makes queries faster
					* durability: keep memory db or other alternative
				* keep a list of all IDs for posts matching given keyword as a list in Redis
				* posts made, break them apart into keywords and append post ID to each keyword
				* issues:
					* post id lists can get large for common keywords
					* lots of key writes for every post since a post could have a lot of keywords
		* get most recent search reuslts
			* bad: request time sorting
				* grab all post ids for a keyword, look up timestamp, sort in memory
			* issues:
				* post ID can be large for very common keywords
				* causes latency (each result requires a lookup, sorting millions of items at request time adds a lot of latency)
			* better:
				* have multiple indices
				* one sorted by creation time and one sorted by like count
				* creation time: standard list since we append to the end
				* like index: have a sorted set, keeps list of items ordered by a score the same way priority queue or sorted list works
				* new post made: add it to indices for every keyword it contains
					* like event happens: update score in sorted set for like index
				* lots of likes happen, so this requires updating many scores so like indices are up to date
					* add stress to system
* potential deep dives:
	* how to handle large volume of requests from users?
		* two requirements make job easier
			* no personalization everyone gets same results
			* 1 minute before post needs to show up
		* use caching:
			* good: distributed cache
				* cache stores most recent results for a given search query
				* when search is performed, service first checks if query in cache
				* if so, return results
				* if not, perform search and store results
				* ensure stale results don't stick around w/ low TTLs
			* better:
				* use a CDN cache that caches at the edge near the user
				* on response to `/search` endpoint, add `cache-control` headers
					* tells CDN when and how long to cache result
					* w/ this:
						* hit search service: go through geographically local CDN host and w/ a hit, results are in 10s of milliseconds
						* w/ distributed cache: go to API gateway, search service, call to search cache, and backthrough
							* 100 ms
	* how to handle multi-keyword, phrase queries
		* good:
			* look at set intersection between every keyword in our query
			* for a query sorted by likes
				* request post IDs for each component
				* intersect them
				* get contents from all IDs and ensure a contiguous string exists
				* return posts passing in filter in order
			* issues:
				* Post ID set for each keyword can be super large 
				* so intersecting it is hard because thats a lot of data to pass over network
				* if keywords heavily overlapping, keywords don't appear next to each other, filtering a lot of is computationally intensive
		* great:
			* bigrams/shingles
			* create tokens for pairs of words
			* these get inserted into likes and creation indices
			* allows for multiple words to be queried (directly access the bigram)
			* issues:
				* increases size of indices (# of bigrams in sentence is linear w # of words, but bigrams are more unique)
				* only extract likely bigrams but fallback to intersection when no entry, but this introduces complexity and challenges
	* large volume of writes:
		* post created, ingestion service tokenizes post (break into keywords) and then write postId to creation/likes indices
		* add capacity to ingestion service and partitioning incoming requests
		* log/stream like Kafka, fan out creation to multiple ingestion instances and partition load
		* buffer requests to handle bursts of post creation
			* queues
		* shard indices by keyword (a thorough c get a shard, d through h..)
			* scale out writes across many instances
		* so we have posts etnering a kafka stream w/ partitions
			* each partition is based on hash of postId or userId (spread work evenly)
			* multiple ingestion instances process in parallel: split into words and then add postIds to that
			* then we write to redis instances that have partitionIds based on the words
		* likes:
			* happen more often than post creaiotn
			* good: aggregate likes, do a batch write
				* instead of writing 500 times for 500 likes, have 1 update w/ an increment of 500
				* requires a secondary batcher service that accepts like events and aggregates them over a fixed window before writing them back to be consumed by the ingestion service
				* issue:
					* improves volume on like volume of most viral posts, but doesn't reduce volume of writes overall since most posts aren't viral
			* great:
				* update like counts in indices if specific milestones
				* 1 like, 2 likes, 4 likes
				* this means index is inherently stale, but data can't be completely trusted
					* ordering is approximately correct tho
				* but this is fine:
					* we get the top N * 2 posts, for each post we query Like Service for most up to do date and then sort based on the fresh count
				* this is a two architecture approach, approximately correct service backed w/ more expensive reranking
				* issues:
					* more complex than others, require more engineering effort
					* changes semantics of like counts in elastic search
	* how to optimize storage on system:
		* caps/limits on each of inverted indices
			* don't need 10 million posts contained
			* keep index up to about 10 thousand items
		* most keywords won't be searched often or at all
			* batch job removes less frequently accessed keywords to cold storage
		* determine what keywords are accessed in past month rarely
			* move indices to blob storage
			* when index needs to be queried, we query redis 
				* if not there, index blob storage w/ small latency penalty
		* 