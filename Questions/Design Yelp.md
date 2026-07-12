- core requirements:
	- functional:
		- users should search businesses by name, location, and category
		- users should view businesses and reviews
		- users should leave reviews on businesses (mandatory 1-5 star rating and optional text)
	- non-functional requirements:
		- low latency for search operations
		- highly available, eventual consistency
		- should handle 100M daily users and 10M businesses
	- constraint:
		- each user can only leave one review per business
- core entities:
	- business: an actual business/service on yelp
		- has name, location, category, average rating
	- user: Yelp user that can search for businesses/leave reviews
	- Review: review a user leaves for a business, including rating and optional text
- api:
	- search for businesses
		- `GET /businesses?query&location&category&page` -> `Business[]`
	- view business and reviews;
		- `GET /businesses/:businessId -> Business & Review[]`
		- to enable pagination can split business and reviews into to endpoints:
			- `GET /businesses/:businessId` -> `Business`
			- `GET /businesses/:businessId/reviews?page=` -> `Review[]`
		- leaving a review:
			- ```
			  POST /businesses/:businessId/reviews {
				  rating: number,
				  text?: string
			  }
			  ```
* high level:
	* users should be able to search for business:
		* client sends `GET` request to `/busineses`
		* API Gateway routes this to the Business Service
		* Business Service queries the Database
		* results returned to the client through API Gateway
	*  users should be able to view businesses:
		* route requests to Business Service
		* Query Database for business details and reviews
			* reviews are kept in the same database and the businesses, but need to be able to join the two tables
		* keep the business searching/viewing logic in the same service because they are closely related
	* users should be able to leave reviews on businesses
		* separate into different service b/c usage pattern is different
		* users search/view a lot, but hardly leave reviews
		* user leaves a review:
				* `POSTS` to `businesses/:businessId/reviews` w/ review data
				* API Gateway routes to Review Service
				* Review Service stores it in database
				* client gets a confirmation
		* are all microservices supposed to have their own database?
			* not necessarily, simpler in context where systems use the same database for multiple purposes 
* deep dives:
	* efficiently calculate/update average rating for businesses to ensure available in search results?
		* worst solution:
			* naively calculate average on the fly
			* `SELECT AVG(r.rating) as average_rating`
			* performance degradation:
				* as the number of reviews grows, query gets more and more expensive
				* unnecessary recalculation, average rating will get calculated even if no new reviews have been added
				* makes query heavy
		* better:
			* periodic update w/ cron job
			* every defined time frame (once a day, hour, etc.), cron job queries reviews table, calculates average rating, updates new average_rating column
			* main downside:
				* doesn't account for real time changes in reviews
				* business w/ few reviews, if you give good rating, average rating should be expected to boost immediately
				* but doesn't happen
		* best:
			* use a synchronous update 
			* add new column to Business table `num_reviews`
			* update average rating, calculate `(old_rating * num_reviews + new_rating) / (num_reviews + 1)`
			* this causes a new problem
				* what if multiple reviews come at the same time for the same business
				* once would get lost due to a lack of synchronization
			* how to solve this?
				* optimistic locking:
					* first read the current state and then update it
					* if state has changed since we read, update fails
					* state has changed: we just check if the # of reviews has changed
					* so if user1 and user2 update at the same time, user 2's update would have failed since number of reviews changed
					* so then we read and recalculate
			* message queue not needed
				* this is because the write ratio is not as a large as the read ratio
	* how can a user only leave only one review a business
		* user only leaves one review
		* bad:
			* application level check
			* check whether the user has reviews business in application layer
			* issues:
				* not robust to changes
				* other services can also write reviews
				* in these cases, application-layer constraint may get violated
				* adds a race condition:
					* same user submits two reviews at once, both pass check and end up w/ two reviews, violating constraint
		* great:
			* add a unique constraint on `user_id` and `business_id` fields
				* ```
				  ALTER TABLE reviews
				  ADD CONSTRAINT unique_user_business UNIQUE (user_id, business_id)
				  ```
			* impossible to violate constraint because database simply refuses to insert duplicate (get an error)
			* write attempt gets processed second and throws a unique constraint error
	* how to efficiently search:
		* database would have a full table scan, checking every record against conditions
		* queries where longitude/lattitude are searched through `>/<` and strings are compared through `like` are slow
		* solutions:
			* add basic index on latitude, longitude
				* B tree bounding index
				* doesn't work as effectively because B tree index isn't optimized for querying dimensional data
					* range queries on 2d data aren't good w/ B tree indexes
					* B tree indices don't understand spatial relationship between points
				* need specialized indexing structures for spatial data
					* R-Trees, quadtrees, geohash-based indexes
			* use elastic search:
				* would support location search through geohash, quadtrees, R-trees
				* name: search by name, use full text search index through inverted indexes (see what documents/entries contain words)
				* has various indexing strategies like geohashing, quadtrees, R-trees
				* issue a single search query in JSON format to ElasticSearch and that will combine all filters and return ranked list of busineses
					* ```
					  "query": {
						  "bool": {
							"must": [
								"match": {
									"name": "coffee"
								}... have other categories for distance, and term
							]  
						}
					  }
					  ```
				* adds complexity because Elasticsearch isn't a primary database
					* not optimized to handle transaction data integrity w/ full ACID compliance or to handle complex transactions
					* best utilized for search and analytical operations across large datasets 
				* to maintain Elasticsearch in sync w/ primary database
					* so you need Change Data Capture mechanism to get changes from primary database and apply to Elasticsearch
					* all DB changes get captured and written to queue or stream and then consumer process reads and applies changes to Elasticsearch
			* Postgres w/ Extensions
				* Don't even bother adding a search-based extension
				* Postgres has PostGIS extension to index/query geographic data
				* Another extension called `pg_trgm` used to index, query text data
				* this creates creates geospatial index on latitude/longitude columns and full text search index on business name/description columns without adding new services
				* scaling not that deep, `10M busineses * 1kb each = 10GB` + `10M busineses * 100 reviews each * 1kb = 1TB`
			* postgres might be too easy
				* mention correct geospatial indexing strat (geohashing vs quadtrees)
				* use quadtrees because updates are infrequent and businesses and clustered into dense regions
				* second pass filtering:
					* get results of query
					* further filter by exact distance
					* calculate distance between user's lat/long and business lat/long, filtering based on desired radius
						* done w/ Haversine formula (Pythagorean theorem for calculating distances on sphere)
				* want to apply distance restriction first since its the most restrictive
				* then apply other smaller set of businesses to apply filters
	* handle searches through predefined location names like cities/neighborhoods
		* can't just use simple radius from central point because cities/neighborhoods aren't perfectly circular
		* need to define a polygon for each location and check if any businesses are within that polygon
		* use GeoJSON to store geographic data and include functionality for working w/ polygons
		* need way to go from location name to polygon, use polygon to filter a set of businesses and exist within it
		* add new table that maps location names to polygons
			* columns for `name`, `type` (city, neighborhood), `polygon`
			* populate this table w/ data from chosen sources
			* index name for efficient lookups
		* how to filter businesses when polygon
			* Postgres and PostGIS have functionality for working w/ polygons
		* add new `geo_shape` field to business documents and use it to find businesses that exist within polygon
			* doing bounding box search on every request is inefficient
			* pre-compute areas for each business and store them as a list of location identifiers in business table when we create
			* so a document in Elasticsearch is:
				* ```
				  {
					  "id": "123",
					  "name": "Pizza Place",
					  "location_names": ["bay_area", "san_francisco", "mission_district"],
					  "category": "restaurant"
				  }
				  ```
			* then do an inverted index on this where the location_names become the keys
			* allow for quick retrieval