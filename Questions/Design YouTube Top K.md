- assume we have large stream of views on YouTube, how to query precisely top K most viewed videos
- functional requirements:
	- be able to query time in a certain time period
	- two types of windows:
		- sliding windows
			- 1 hour is the time between T - 1 and T
			- current time is 10:06, last 1 hour is time between 9:06 and 10:06
			- exact time
		- tumbling windows
			- rounded to the nearest full hour and boundary
			- 10:06 goes to 9:00 to 10:00
		- sliding windows are significantly easier to support
	- core requirements:
		- query top K videos of all time
		- query tumbling windows 1 hour, day, month
- nonfunctional requirement:
	- 1 minute buffer b/w view and when it gets tabulated and included
	- system should be precise (for now)
		- precomputations can allow us to achieve this latency
	- return results in 10s of MS
	- handle a lot of views per second
- approach:
	- system can calculate top k vids of all time initially
	- extend to support time windows
- core entities:
	- video: what we are watching
	- view: event that happened where some watched it
	- time window: associated w/ query all time, last hour, last day, last month
- api, system interface:
	- endpoint to retrieve top K videos
	- `GET /views/top-k?window={WINDOW}&k={K} --> {videoId: string, views: number} []`
	- pagination not really a issue since we are capping the number of videos
- high level design:
	- build something, then optimize it
	- clients need to query the top k videos for all time
		- counters for each video, query top videos from a list
		- assume there's another Kafka stream of view events from which we consume
		- assume there's a `ViewEvent` topic thats partitioned by video ID, simple consumer service views events and updates Postgres w/ results
		- so:
			- consumer retrieves vew event
			- updates counter for video id
		- acknowledge that there's a lot of writes to a single instance
		- top k service: queries the postgres database w/ an index on the views
			- ```sql
			  select "videoId", "views",
			  from VideoViews
			  order by "views" desc limit K
			  ```
			* o(k) operation
			* updating an index: O(1) appned to a O(log n) index update
	* query tumbling windows of 1 {hour, day, month} and all time
		* table schema includes timestamp column
		* hour of a view event
		* one row for each video that viewed at least once a given hour
			* so many rows for videos viewed multiple times over hours
		* just adjust query:
			* add index on timestamp
			* ```sql
			  select "videoId", SUM("views") as "views",
			  from VideoViews
			  where "timestamp" >= {windowStart} and "timestamp" <= {windowEnd}
			  group by "videoId"
			  order by sum("views") desc limit {k}
			  ```
			* would require a lot of scans (go through every single row)
			* processing billions of rows take a while so this needs to be optimized
* deep dives:
	* cut down on the # of queries to the database
	* we have a minute grace period from when video view happens to when it needs to be tabulated
		* so we can precompute and cache
		* good solution: cache top K for each window
			* put a distributed cache like redis/memcached
			* if request in cache return immediately
			* request not in cache, compute it and store in cache and return
			* store cache entries w/ `top-k:{window}:{truncated_timestamp}`
			* issue:
				* cache expires, requests flood DB
					* use a queue for requests
				* break SLA because top K takes longer than 10 seconds of milliseconds
		* great solution:
			* precompute the top K for each time window w/ a cron job
			* doesn't break SLA when cache expires, cron job running at fixed intervals, can get ahead of expirations
			* issues:
				* adds operational complexity
				* what if job fails
				* retain cache entries for a couple hours where pre-computation is late (top K just serves stale data for a short period)
	* how to handle massive # of writes to the database
		* do estimation (we need it to influence design)
		* `70 B views/day / 100k seconds/day = 700k writes per second`
		* modern RDMS can do 10k+ writes per second per node
		* storage: 
			* `1 hour content/second / (6 minutes content/video) * (100k seconds / day) = 1M videos/day`
			* `total videos = 1M videos/day * 365 days/year * 10 years = 3.6B video
			* storage: `4B vidoes * (8 bytes/id + 8 bytes/count) = 64 GB`
		* partition database based on partitions of `ViewEvent`
			* `VideoId`based partition for the ViewEvent topic
			* so consumers read from a topic and write to the same shard database
			* view comes in, assigned to a shard based on video id
			* view consumer for that partition reads event, fires off a write to the database for that shard
			* issue: need 70 instances to shard to
				* also need to query each shard and merge results 
			* top K from each shard, guaranteed to have global top K in our final list
		* batching ingestion:
			* lots of views happen on a small # of videos, 3.6B videos, lot of those views are on small # of popular videos
			* batch writes for each video and flush periodically to database
			* use Flink: stream processing framework that gives convenient tools for batching and aggregation in streaming applications
			* BoundedOutOfOrderingStrategy for late events:
				* ok waiting for some time for late events, use a tumbling window to aggregate views
				* flink reading from Kafka: host fails, Flink rewinds checkpoint offset in Kafka topic and resume processing from there
				* flink job: causes big lump of writes every hour
					* spread across shards, acceptable and optimal
				* the number of writes will be smaller than before, summing many views that happen in on hour
				* by batching, we can have more shards
	* efficiently do top K query
		* when we don't have cache, we query database and sum all views for each video in the window
		* then pick top K
			* look at SQL query
		* solutions:
			* aggregate: track views of a video for each day
			* query for larger windows through aggregates
			* periodically query database, aggregate total # of views for the higher granularity, group hours views by day or days by month, store to new table
			* additional tumbling windows for Flink (aggregate longer periods)
			* issues:
				* cron read/write to db means additional load hard to sustain
				* top k is delayed by writing hour-grain aggregates, kicking off cron, aggregating hours with other hours, writing them, writing to DB, reading back
			* great: maintain aggregates for each window:
				* VideoViewsLast {Hour, Day, Month, All Time}
				* video view for last month:
					* Flink jobs aggregate at hour grain views for each video
					* new hour passed ready to write to database, make updates to window tables (`VideoViewsLastHour`) instead of writing to hour-grain `VideoViews` table.
				* issues:
					* move complexity from top-K reads to writes
					* have to write to 4 tables instead of 1
						* optimize these operations by using unlogged tables 
						* alter fsync
						* use benchmarks
			* also great: keep aggregates in Flink instead of reading/writing to Views DB
				* move state off of the heap onto disk so it scales with large size of data
				* top K calculations performed in Flink natively
					* write output to cache at conclusion of every minute instead of a top K cron
				* rolling window aggregator keeps in memory view count for each window for each video
				* window complete, emits VideoWindowCount containing window/count of views
				* top k aggregator keeps in memory top K videos for a window
				* this is cumbersome and relies too much on the Flink aspect
	* support sliding windows:
		* when a batch of views for the most recent minute have come in, we can increase views that came in by that minute and decrease by views that happened 60 minutes ago for hours
		* flink job aggregates at a minute grain
			* new minute has passed, ready to write to DB
			* read all views happened in the minute 60 minutes ago
			* write all of the views that happened to video views
			* update VideoViewsLastHour with the difference between the increment and decrement
			* do this for all the windows, decrement is always 0 for all time window
		* need minute grained data for a whole month so we can decrement it when we expire out of last month window
			* wasteful so support sliding windows for last hour and tumbling windows for the last day and last month
		* can also have a consumer group increment counts and the other decrement them when they have expired
	* preciseness:
		* do we need to be exactly precise?
		* use count-min sketch: set of hash functions that maps items to 2d array of counts
		* forgets items, but remembers how many times they've been seen by their hash
		* maps items to a 2d array of counters
		* forgets actual items but sees them in terms of their hash
			* add to CMS (update sketch to remember the view)
			* estimate # of times we've seen it to serve as an upper bound on the # of views item has received and decent approximation
			* add count to a heap: truncate list periodically to not use excessive memory
			* then get top K
		* what data structure can we use to keep sketches and sorted lists
			* Redis: make a view event consumer, have each view event trigger a CMS.INCRBY than a CMS.QUERY
				* ZADD the items to our sorted set, keep sorted set trimmed to a 1000 entries
				* Durability is a concern when we introduce Redis
				* each view event mutates sketch, issue of durability
				* we can lose data if view data is added to sketch but not updated in sorted set
				* rebuilding from Kafka is an option but its slow
			* Flink based approach:
				* CMS and sorted list can be maintained in Flink's state management in memory (checkpointed by Flink to avoid losing state)
				* approach is more resilient to failures than Redis and more efficient since data structure is in memory
			* works well for tumbling windows only because we are adding to our sketch
				* sliding window: have to subtract, redis won't let us
		* types of databases:
			* influxDB or prometheus (time-series engine w/ downsampling)
				* ingest view counts through kafka, write per-minute points are keyed by time with videoId and use tasks/continous queries for downsampling (top K)
				* billions of distinct videoIds, top() queries are doing scans (too much cardinality)
				* not good for querying eveyrthing, only a single (just a graph)
			* timescale DB (Timescale DB, Postgres + hypertables + continuous aggregates):
				* have hypertables with time-based partitioning w/ time series features
				* model per-hour aggregates as a hypertable partitioned by hour-grained time and videoId
				* index w/ views column to further improve performance of the query
				* continuous aggregates to maintain rollups for last hour/day/month
					* save queries w/ `ORDER BY "views" DESC LIMIT {k}`
				* challenges:
					* lots of sharding
			* real-time OLAP:
				* ingest from Kafka, roll up per-minute by `videoId`, query w/ topN groupBy over a time filter
				* pre aggregate data data on ingest
				* Druid: ingestion-time rollup groups by a time bucket and videoId, apply count aggregators so segments store pre-summed rows
				* Pinot: star-tree indices and materialized views pre-aggregate metrics for fixed patterns like topK video id over a window
					* star-trees prune most data and serve limit K quickly
				* ClickHouse:
					* use a materializing view (summing merge tree / aggregating merge tree) to maintain per minute/hour/day rollups on ingest (put pressure on writes), queries for fixed windows read the rolled-up table and do order
				* issues:
					* system can get flaky at scale in production
					* node outages and compaction can cause issues
				* There's NO POINT IN USING THIS UNLESS UVE USED AT WORK