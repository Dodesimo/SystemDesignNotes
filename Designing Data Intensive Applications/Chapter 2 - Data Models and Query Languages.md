- abstractions allow for APIs to be built upon APIs
- sql:
	- relational model:data in relations, where relation is unordered collection of tuples
		- relation = table
		- tuple: row
- no sql:
	- handle larger datasets, high write throughput
	- free open source
	- specialized query operations not supported by relational DB 
- polyglot persistence: multiple things exist for differing use cases
- mismatch exists between objects and relational tables
	- convert raw bytes of tables, rows, columns into objects
- given example of resume:
	- each person has exactly one first_name and last_name, so can be columns in the user table
	- but most people had more than one job, varying numbers of periods, number of pieces of info
		- one to many relationship
		- put positions, education, contact_info in separate tables w/ a foreign key reference to users table
		- so when you join on that table you get a list of values
	- later versions of SQL: add support for structured data types and XML data
		- so you can have multi-valued data in the same row
	- encode the jobs, education, contact info as JSON/XML document, store in a text column, and have application logic that parses it
- JSON: reduces impedance mismatch between application code and storage layer
	- lack fo a schema, which could hurt data encoding
- representation also has better locality
	- for a profile fetch, need to perform a bunch of queries/joins across bunch of tables
	- JSON, all the data is in the right place so on query is enough
- many to one and many to many relationships:
	- why do you want the region id and industry ID to be given as IDs instead of plain text strings?
		- have consistent style, spelling
		- avoid ambiguity
		- name is stored only once so easy to update across
		- translation: allows for the standardized lists to be localized
		- search can be done better through encoding
	- ids are generally better to avoid deducation and denormalization
	- with this many to one relationship (since many people working in the same region has the same id for location), document model doesn't fit that well
		- this type of information suggests join, and document DB have little support for joins 
- so document dbs work well for one to many relationships (since you can have nested JSON structures)
	- but makes many to many relationships hard and doesn't support joins
	- so you have to either duplicate data or manually resolve references from one record to another 
- network model:
	- generalization of hierarchical model
	- record can have multiple parents
	- for region, every user who lives in that region is linked to it
	- how to find a record: follow path from root along chain of links
	- traversal of a linked list: start at head, look at one record at a time until one we find
	- with many many relatinoships do a DFS or something
- relational model:
	- all data is in the open
	- collection of tuples (rows)
	- no complicated nested data structures
	- can insert a row into a table without worrying about foreign key relationships to and from other tables
	- query optimizer decides what parts of query executes in what order and what indices to use
	- query data in new way: declare new index, queries automatically use  new index
- how do both relational and document dbs represent many to one and many to many relationships?
	- related item is referenced through a foreign key in relational model or document reference in document model
- why use document DB?
	- for some applications closer to data structure used, schema flexibility, and better performance due to locality
- relational model:
	- provides more support for joins, many to one, many to many
- application data has document-like structure (one to many relationships), use document model
	- shredding: splitting a document into multiple tables can be cumbersome and unnecessary hard
- document model has limitations, can't directly refer to nested item but need to have an access path
	- poor support for joins could be a problem if there are many to many relationships
	- reduce the need for joins through denormalization, but would require maintaining consistency
	- can emulate joins by making multiple requests to the DB, but moves complexity into application and slower than a join
- most document DBS don't have schema on the data
	- means that arbitrary key/values can be added to document and when reading, no guarantee what fields the document may have
	- schema on read: the structure of data is implicit, interpreted on read
	- rdbs: schema on write, schema is explicit and database ensures all data conforms 
- why does schema matter?
	- when you want to change data format
	- currently storing full name in a field, change it to store first name and last name separately
	- for document DB: just start writing new documents and have code that handles when old documents are read
		- ```
			  if (user && user.name && !user.first_name) {
			  user.first_name = user.name.split(" ")[0];
		  }
		  ``` 
* statically typed database schema:
	* migrate through:
		* ```
		  ALTER TABLE users ADD COLUMN first_name text;
		  UPDATE users SET first_name = split_part(name, ' ', 1) // existin column, split, take first index
		  ```
		* my SQL: when altering table, could take hours of downtime when adjusting a large table
			* quicker update: just leave the field by default null, and fill at run time (but that makes it a document db)
* schema on read:
	* advantageous if data is heterogeneous (items don't all have the same structure)
		* many different types of objects, can't put in its own table
		* structure of data is determined by external systems, have no control, can change at any time 
* documents: 
	* stored as single continuous string (JSON, XML, binary)
	* access entire doc, performance advantage to this storage locality
		* split across multiple tables, multiple index lookups are required for retrieval (more disk seeks and take more time)
	* this advantage doesn't make sense when you access a small portion (wasteful on large docs)
	* updates to a doc, entire doc gets rewritten
		* only time rewrites doesn't happen is when modifications don't change the encoded size of the document (can be easily done in place)
		* recommended to keep document fairly small and avoid writes increasing the size
* some relational databases allow for grouping related data for locality:
	* Spanner DB: schema declares that table's rows should be nested
	* Oracle: multi-table index cluster tables
* convergence of document and relational databases:
	* most relational DB (not MySQL) allow for XML support (local modifications, index/query XML)
	* PostgreSQL and IBM DB2 have similar levels of support for JSON documents
	* document DBs support relational-like joins
		* RethinkDB and MongoDB drivers resolve database references (like unoptimized client-side join) 
* query languages:
	* SQL: declarative language
	* IMS/CODASQL: imperative
	* imperative:
		* ```
		  sharks = []
		  for a in animals:
			  if a.family == "sharks":
				  sharks.append(a)
		  return sharks
		  ```
	* declarative:
		* relational algebra
			* ```
			  sharks = sigma_(family == "sharks") (animals)
			  ```
			* selection operator, return animals that match the condition family
	* imperative language: tell computer to perform operations in a particular order (go through line by line)
	* declarative query language: specify pattern of the data, what conditions/results to meet, how you want it transformed (aggregation, sorted, grouped)
		* but you don't specify how 
	* declarative is attractive because its more concise and easier to work with, hiding implementation details
	* SQL: doesn't guarantee any ordering so doesn't mind if order changes
		* if the query is written as imperative code, database not sure if the code is relying on ordering or not
	* imperative is hard to parallelize across multiple cores and multiple machines because you are specifying instructions need to be performed in a particular order
	* declarative languages: easier to do parallel execution, because you only specify the pattern of the results, not the actual algorithm
	* declarative:
		* specify pattern of data:
			* CSS selector
				* ```
				  li.selected -> p {
					  background-color: blue;
				  }
				  ```
			- specify the pattern of the data, now hot to do it (just specify all list items with select flag and their paragraph to be blue)
	- imperative: 
		- iterate through every single object that is an li, go through all the children if it has selected class name, see if it has a tag of p, set the attribute
		- literally telling the computer how to do it
		- harder to understand 
		- also:
			- if selected class is removed, the color isn't removed even if the code is reran
			- CSS: browser detects when li.selected -> rule doesn't apply
			- because you specify the operations in the loop to lookup tags, if there's a new API that has an improvement, you would have to adjust the code
				- since the actual implementation details are hidden in declarative it can automatically utilize these new API calls. 
	- map reduce:
		- logic of query expressed through code snippets (called by the processing framework) repeatedly
			- based on map (collect) and reduce functions in many functional programming langauges
		- take all sightings, express with:
			- ```
			  SELECT data_trunc('month', observation_timestamp) as observation_month, sum(num_animals) as total_animals
			  FROM observations
			  WHERE family = 'Sharks'
			  GROUP BY observation_month
			  ```
		* map reduce:
			* ```
			  db.observations.mapReduce(
				  function map() {
					  var year = this.observationTimestamp.getFullYear();
					  var month = this.observationTimestamp.getMonth();
					  emit(year + "-" + month, this.numAnimals);
					  // for every document that matches the query (which is sharks), call this function.
				  },
				  function reduce(key, values) {
					  return Array.sum(values); 
				  },
				  // the above map function emits key (year and month), and value (which is the total number of animals). this reduce function then is called once for all key-value pairs that share the same key (so you sum all the values and the values are the number of animals). final output is stored in the collection monthlySharkReport
				  {
					  query: {family: "Sharks"},
					  out: "monthlySharkReport"
				  }
			  );
			  ```
		- map and reduce must be pure functions that only use the data that is given to them and nothing else
		- MongoDB has a similar declarative query language called an aggregation pipeline
			- more JSON like structure
	- application with mostly one to many relationships or no relationships between records, document model is best
	- complex many to many relationships: use a graph database
	- property graph:
		- each vertex has a unique id, set of outgoing edges, set of incoming edges, collection of properties (key, value pairs)
		- edge: has a unique identifier, vertex where the edge starts (tail vertex), vertex where edge ends (head vertex), label that describes relationship between two vertices, collection of properties (key-value pairs)
		- two relational tables
		- ```
		  CREATE TABLE vertices (
			  vertex_id integer PRIMARY KEY, 
			  properties json
		  );
		  
		  CREATE TABLE edges (
			  edge_id integer PRIMARY KEY,
			  tail_vertex integer REFERENCES vertices (vertex_id),
			  head_vertex integer REFERENCES vertices (vertex_id),
			  label text,
			  properties json
		  )
		  
		  CREATE INDEX edges_tails on EDGES (tail_vertex) # so we can easily query tail vertices 
		  CREATE INDEX edges_heads on edges (head_vertex)
		  ```
		- so because a vertex can have edges connecting it to any other vertex (no schema restrictions)
			- since the edge vertices have indices, you can traverse graph by following chain of vertices
		- can easily extend graphs with other vertices that are heterogeneous
			- so accomodate changes
		- cypher query language:
			- ```sql
			  CREATE -- create objects that have attribute and then additional properties
					  (NAmerica:Location {name:'North America', type:'continent'}),
					  (USA:Location {name:'United States', type:'country' }),
					  (Idaho:Location {name:'Idaho', type:'state' }),
					  (Lucy:Person {name:'Lucy' }),
					  (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica)
					  -- create edges that have labels titled "within" 
			  ```
		* how to query this information:
			* ```MATCH
			  (person) -[:BORN_IN]-> () -[:WITHIN*0]-> (us:LOCATION {name:'United States'})
			  (person) -[:LIVES_IN]-> () -[:WITHIN*0]-> (eu:Location {name:'Europe'})
			  RETURN person.name
			  ```
			- find all vertices that have outgoing BORN_IN edge to a vertex that can then be traced to within United States AND they have a outgoing LIVES_IN edge that can be traced back to Europe
			- for all of these vertices return the name property
			- because this is declarative, you don't need to specify execution details because the query optimizer chooses for you
		- instead of using graph query language, can you query a graph in SQL?
			- harder because when doing relational queries, you need to know in advance what joins are required
				- in graph query, need to traverse variable number of edges
				- Cypher handles this with `:WITHIN*0`: follow WITHIN edge zero or more times
				- this variable nature is expressed through a recursive common table expression  (WITH RECURSIVE syntax)
					- 30 lines of query code for the same Cypher, essentially get set of vertex ids for in_usa, in_europ,, in_usa, lives_in_europe,  and then join to find people born in US and live in europe
		- triple stores:
			- all information is stored in subject, predicate, object (RDF triples)
			- subject is a vertex in the graph
			- object can be a primitive data type (string, number)
				- so in that case, the predicate and object are equivalent to a key, value of a property on the subject vertex 
					- `(lucy, age, 33) -> lucy: {"age": 33}` (essentially a property)
			- object can be another vertex in the graph, predicate is an edge in the graph, subject is a tail vertex + object is the head vertex
			- can write in a format called Turtle
				- vertices are written in `_:someName`
				- when subject is an edge, the object is a vertex: `_:idaho :within _:usa`, `_:usa` is a vertex
			- predicate is property, object is string literal so (`_:usa :name "United States")
		- semantic web:
			- publish website information as machine readable data for computers to read
			- RDF (resource description framework): mechanism for different websites to publish data consistently
			- can express in XML
				- have WITHIN or LIVES_IN have a specific URI to avoid name space collisions and allows for merging with other data
		- SPARQL: query language for RDF data model
			- ```
			  PREFIX : <urn:example:>
			  SELECT ?personName WHERE {
				  ?person :name ? personName,
				  ?person :bornIn / :within* (go as many within edges) / :name "United States".
				  ?person :livesIn / :within* / :name "Europe"
			  }
			  ```
			- because RDF doesn't distinguish between properties and edges (uses predicates for both), use same syntax for matching properties and edges
		- data log:
			- exploits parallelism
			- data model is similar to triple store, but instead written as predicate(subject, object)
			- done through cases:
				- ```
				  within_recursive(Location, Name) :- name(Location, Name) 
				  // this is saying that if the location has name return true
				  within_recursive(Location, Name) :- within(Location, Via), within_recursive(Via, Name). // this is saying that anything with an intermediate that resolves to the name should also be true
				  migrated(Name, BornIn, LivingIn) :- name(Person, Name), born_in(Person, BornLoc), within_recursive(BornLoc, BornIn), lives_in(Person, LivingLoc), within_recursive(LivingLoc, LivingIn)
				  ?- migrated(Who, 'United States', 'Europe')
				  ```
