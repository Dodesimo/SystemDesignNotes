- functional:
	- aggregated feed of news articles fro thousands of source publishers
	- scroll through feed infinitely
	- click on articles, redirect to publisher's website. 
- non functional:
	- prioritize availability over consistency
	- scalable to 100 million DAU w/ spikes up to 500 million
	- low-latency feed load times (< 200 ms)
- core entitities:
	- article: news article w/ attributes like id, title, summary, thumbnail, URL, publish date, ID, region, media URLs
		- core content entity
	- publisher: news source w/ attributes like id, name, url, feed URL, region
	- user: attributes w/ id and region (can be inferred from IP or explicitly set ways)
		- can be tracked even for anonymous users
- api:
	- get a aggregated feed of news articles:
		- ```
		  GET /feed?page={page}&limit={limit}&region={region} -> Article[]
		  ```
	* for users to view specific article, no API endpoints since browser navigates to publisher website once article clicked
* high level design:
	* view aggregated feed of news articles from thousands of source publishers all over
	* need to collect content from publishers and serve it efficiently to users
	* data collection service: background process that gets content from thousands of news sources
		* Polls publisher RSS feeds and APIs every few hours based on update frequency.
		* RSS: simple format allowing publishers to syndicate content to other websites and readers
			* XML
		* Works over HTTP: make a GET request to the RSS feed URL to get content, response is XML document w/ article title, link + metadata
	* Publishers: thousands of news sources worldwide providing content through RSS feeds/APIs
	* Database: contains collected articles, publishers and metadata
	* Object storage: stores thumbnails for the articles
	* data collection service:
		* queries database for list of publishers and their RSS feed URLs before querying each one after another
		* extract the content, metadata, and downloads to use thumbnails
		* store thumbnail files in Object Storage and save article data w/ media URLS in database
	* why download the content in our own object storage?
		* serve images to users quickly and efficiently, not rely on publisher's servers that may be slow or down entirely
		* standardize quality/size of images to ensure consistent user experience
	* Feed Service:
		* handles user requests
		* Client: allows users to interact w/ Google News through web browser or mobile apps 
		* API Gateway: we know what this is
		* Feed Service: handles user feed requests through querying relevant articles based on user region and formatting response for consumption
	* why is the feed service and data collection service separated?
		* feed service is read-heavy and data collection service is write heavy
		* also have different update frequencies (real-time, batch)
		* based on this need to be scaled differently
	* when user wants a news feed:
		* send a GET request to the feed endpoint
		* route the request to the Feed Service
		* query DB for recent articles in user's region through publish date
		* database returns article data and media URLs in object storage
		* Feed Service formats response, returns to client through API Gateway
	* scroll through feed infinitely
		* add offset-based pagination through page numbers and page sizes
		* user loads feed:
			* GET request to first page `/feed?region=US&limit=20&page=1`
			* feed service queries for first 20 articles in the region ordered by publish date
			* response: has articles plus pagination metadata, total_pages and current_pages
			* stores current page number for next request
		* user scrolls and goes to the end of current content:
			* send GET request to `/feed?region=US&limit=20&page=2`
			* calculate offset (page - 1 * limit) and fetch next 20 articles
			* database query fetches articles w/ OFFSET and LIMIT clauses
			* process repeats as user continues scrolling through pages
	* click on articles:
		* the browser does this for us 
		* Google News doesn't actually host any content (just point to publisher's website)
		* for analytics, there would be a tracking endpoint that logs click event and then 302 redirect to publisher's site
* potential deep dives:
	* improve pagination consistency/efficiency
		* offset-based pagination approach has limitations when new articles get constantly added
		* this is because offsets are not stable when additions are done
			* results in duplicate articles or missing new content that was published
		* how to add cursors:
			* timestamp based cursors:
				* articles are returned w/ a cursor representing timestamp of last article in the response
				* for subsequent requests, client includes this cursor, query for articles published before timestamp
				* eliminates pagination drift since we query relative to a fixed point (new articles don't impact time of older content)
				* cursor is a stable reference point
				* add index on published_at
				* issue:
					* multiple articles have identical timestamps
					* query for articles before a timestamp, miss articles sharing same timestamp
					* creates gaps where users miss articles published at the same time as cursor boundary
			* better:
				* create a composite cursor that combines timestamp and article ID
				* adds total ordering even w/ identical timestamps, since article ID has a tie-break
				* DB query is now `WHERE (published_at, article_id) < (cursor_timestamp, cursor_id) ORDER BY published_at DESC, article_id DESC, LIMIT 20`
					* uses tuple comparison to maintain composite ordering
					* create a composite index on (published_at, article_id)
				* issues:
					* increased complexity in cursor generation/parsing (just more complex logic)
					* but modern databases are good at handling tuple comparisons efficiently
					* there's some storage cost since cursors are longer, but not that sig. compared to benefits
			* also good:
				* monotonically increasing article IDs
				* use time-ordered UUIDs or database auto-increment IDs that naturally increment w/ each new article
				* since articles are organized chronologically, new articles have higher IDs than older ones
				* now every single article has a distinct ID
					* use that as a key (query of `WHERE article_id < cursor_id ORDER BY article_id DESC LIMIT 20`)
				* eliminates timestamp collisions
				* database just needs a simple index on article_id column
				* most modern systems have ULIDS 
				* challenge:
					* ID generation strategy (migrating to monotonic IDs need careful data migration, also need to ensure ID generation system can handle high throughput without bottlenecks)
					* coordinate ID generation across multiple instances to maintain ordering
		* how to achieve low latency feed requests?
			* billions of feed requests but relatively lesser article writes
			* aggressively cache + pre-compute article rankings
			* solution:
				* Redis Cache w/ TTL:
					* cache recent articles by region in Redis
					* separate cache keys for each region, store articles as sorted sets ordered by timestamp
					* users request feed, check Redis for cached articles and fall back to database on cache misses
					* set a TTL of 30 minutes to ensure cache stays reasonably fresh while reducing DB load significantly
						* high hit rate because every is requested most recent articles
					* issues:
						* cache misses require DB query (still expensive)
						* during cache expiration, everyone hits the database at the same time
							* creates thundering herd problem
						* results in inconsistent user experiences, some requests are fast (cache hits) or others being very slow (cache miss)
				* Real-time Cached Feeds w/ CDC
					* precompute and cache feeds and use Change Data Capture for immediate updates
					* maintain pre-computed feeds
					* when new articles published, CDC updates relevant regional caches without TTL
					* Data Collection service stores new articles in the DB (triggers CDC events)
						* consumed by Feed Generation Workers determine which regional feeds need updates based on article region and relevancy
						* add the new article to the correct Redis sorted set w/ timestamp as score
					* there is no longer a TTL, only maintain the top N articles per feed
						* use `ZADD`, then call `ZREMRANGEBYRANK`
					* to read from the sorted sets, use `ZREVRANGE`or use a secondary cache look up
					* issues:
						* add more complexity w/ CDC infra and message queues and worker processes
						* storage costs increase because w/ duplication, but argue that it enables quicker hosting
		* ensure articles appear in feed within 30 minutes of publication
			* polling RSS feeds every 3 to 6 hours is too slow
				* breaking news needs to be quick
			* solution:
				* increase RSS polling frequency
					* add a tiered polling system where high-priority publishers get polled every 5-10 minutes, medium priorities like 30 minutes, and low priority every 2-3 hours
					* publisher priority DB table maintains this
					* GET requests made to each feed URL, new articles detected, processed by content ingestion pipeline
						* workers track last-modified headers and ETags from RSS responses to avoid processing when feeds haven't changed
					* issue:
						* lot more requests, increases server costs and risks overwhelming publisher servers (blacklist)
						* doesn't solve fundamental limitation that we are reactive instead of proactive
							* breaking news could still take up to 5 mins in system
							* not latency sensitive enough
						* not all have RSS feeds
				* intelligent web scraping:
					* visit news articles, extract articles directly from HTML structure
						* have a database of website patterns and CSS selectors
					* crawler navigates to publisher homepages and category pages
						* looks for new article links (classes like "article-headline", "story-link", "news-item")
						* extracts URLS, titles, publication timestamps
						* each target website has a fingerprint DB of previously seen articles through hashes
						* new articles detected, scraper follows URLS to extract relevant content
						* normalized into standard article format and into same processing pipeline as RSS sourced content
					* intelligent scheduling (high-traffic sites only get scraped 10-15 minutes), smaller sites is hourly
					* issues:
						* web scraping has a lot of maintenance because websites frequently change HTML structure
						* slower than RSS parsing
						* fallback for publishers w/o RSS feeds (have a hybrid approach, RSS when available, scraping when necessary)
				* publisher webhooks w/ fallback polling
					* publishers notify immediately through webhooks when new content is published
					* `POST /webhooks/article-published`: publishers can call containing article metadata or content payload
					* endpoint: highly available, handle traffic spikes (stories break, multiple publishers notify us simultaneously)
						* validates payloads, extracts data, queues content for processing through ingestion pipelines
						* add webhook authentication through API key
					* publishers w/ webhooks:
						* process content within seconds of publication, since the webhook payload has essential metadata that can be used to update cache
					* keep RSS polling and scraping for publishers that don't support webhooks
					* issue:
						* do publishers want external dependencies like webhooks?
				* the best solution is just a mixture of all of these approaches
		* how to handle media content?
			* only need to show thumbnails
			* we get our own copies of publisher images because publisher images are slow, change URLS, or unavailable
			* approaches:
				* never store raw data in database
				* S3:
					* store thumbnails in S3, reference URLs in database
					* during article collection, download original image, generate thumbnail, upload it to S3, store S3 URL in metadata
					* issue:
						* high latency loading thumbnails from distant regions
						* no support for different densities
				* Use CDN:
					* Store thumbnails in S3, serve through CDN for global distribution
					* generate multiple thumbnail sizes use HTML srcset attributes or client side logic to request for appropriate version based on device and screen density 
					* issues:
						* higher storage costs for multiple thumbnail variants, but has a lot of performance gains
						* caching reduces S3 requests by over 90%
		* how to handle breaking news?
			* what helps:
				* news consumption is regional
				* users want local/national news from regional region
				* so we deploy infra close to users in each region, each regional deployment handles content/traffic for that area
			* feed service:
				* horizontally scale w/ auto-scaling groups
				* multiple instances of feed service behind load balancers, auto scale w/ new instances when utilization exceeds thresholds
				* also stateless, so horizontal scaling is very straightforward
			* database layer:
				* cache drastically reduces load on DB
				* all read requests to fetch feed should hit cache, scale challenges offloaded to cache
			* cache layer:
				* Single redis instance: only about 100k requests per second.
				* read replicas: 
					* each Redis master has read replicas to distribute query load
					* each master stores all regional content without complex sharding
				* write operations go to the master
				* read operations for feed requests are load balanced across replicas
					* redis sentinel managers cluster
					* master fails, one replica gets promoted to master
					* less than 200ms for redis, so users won't notice small delays
					* all of these read replicas get updated through whatever qurorum protocol
				* so when there's traffic spikes,
					* spin up read replicas to handle increased load
		* support category-based news:
			* current design supports regional feeds, also want content organized into categories
			* with current architecture, when major sporting event happens and 10M simultaneously request Sports feeds, query DB for sports articles, filter results, and generate responses for each request
			* solutions:
				* bad: database query filtering
					* add category column to Article table
					* modify to filter based on that category
					* add category extraction during article ingestion
					* feed service generation takes in a composite index on three items
						* `(region, category, published_at)`
					* issues:
						* lots of performance problems (direct database operations)
						* can't cache category results because there's a category for each region, so 250 different keys
				* better:
					* pre-compute category feeds in Redis
					* so there's feed sorted sets like `feed:sports:US` and `feed:politics:UK`
						* during article ingestion, Feed Generation Workers categorize each article and update multiple sets simultaneously
						* when article gets published, added to both `feed:US` and `feed:sports:US`
					* limitation:
						* 25 categories across 10 regions, lots of seperate sorted sets
						* each category feed has 1,000 to 2,000 articles
							* adds to redis memory requirements
						* cache invalidation is more complex as articles need to be removed from multiple sorted sets
						* single article belongs to both regional + category feeds
							* so need to do operations across many
							* can impact read performance
				* best:
					* just change metadata in main cache to include category
					* then in memory, filter based on category
					* so we retrieve the entire regional cache, get the most recent 1000 articles, filter this in-memory and select articles where category is equal to selected one
					* still quick
					* minimal changes to architecture, just change to the metadata
					* memory usage is reasonable due to no duplication across multiple caches
		* customization:
			* bad:
				* real-time recommendation scoring
				* track user behavior and preferences, users request feeds,  service scores recent articles against their profile 
				* can also use collaborative filtering to identify patterns like A and B also engage w/ C
				* only articles that match or have high scores are retained
				* issues:
					* destroys latency
					* calculating a lot of scores live
			* good:
				* precomputed feeds in Redis sorted sets
				* background workers continuously update feeds, scoring content relevant to user's interests
				* combine preferences (subscribed topics, preferred  publishers) w/ behavioral signals (clicks, reading time, shares) to build profiles. 
					* articles get published, recommendation workers identify relevant users and add articles to their feeds
				* active users get dedicated caches, inactive users get feeds generated on demand from category
				* issue:
					* dedicated caches for millions of users
					* cache staleness: causes UX problems when we have interests changing rapidly
					* personalized feeds: miss globally breaking news
			* great:
				* dynamic hybrid personalization
				* maintain lightweight preference vectors per user and mix pre-computed category feeds
				* user interested in tech sees feed w/ 60% `feed:technology:US`, 30% `feed:business:Us`, 10% `feed:trending:US`
				* mixing algo uses preference vector to determine best ratios
				* builds on exisitng category cache infra, reduces memory requirements, and we can use ML to balance mixing ratios through engagement patterns
				* issues:
					* reduced personalization compared to a full recommendation engine