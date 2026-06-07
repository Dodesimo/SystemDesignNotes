- functional requirements:
	- view list of coding problems
	- view a given problem, code solution in multiple languages
	- submit solution get instant feedback
	- view live leaderboard for competitions
- nonfunctional requirements:
	- availability > consistency
	- isolation and security running user code
	- return results within 5 seconds
	- support competitions w/ 100000 users
- small system: few hundred thousand users and roughly 4000 problems
- core entities:
	- problem: statement, test cases, expected output
	- solution, user's code submission and result of running code against test cases
	- leaderboard
- api/system interface:
	- view list of coding problems:
		- `GET /problems?page=1&limit=100`: returns a list of partial problems (problem title, id, level, tags/category, doesn't return entire problem or code stubs)
			- paginated 
		- view specific problem: `GET /problems/:id?language={language}`-> Problem
			- query param. for language can default to any language (Python if not provided)
			- allows returning code stub
		- return result of submitting solution to problem:
			- POST endpoint:
				- `POSt /problems/:id/submit -> Submission {code: string, language:string}`
					- pass in problem id as a query param
					- code as a string 
					- language as a string
					- return a Submission: containing result of running code against test cases
		- endpoint for leaderboard:
			- `GET`: returns a ranked list of users based on performance in the competition
				- `GET /leaderboard/:competitionId?page=1&limit=100` returns leaderboard of the first 100 users ranked
		- all data passed w/ JWTS
- high level design:
	- for this one, since its a smaller system, monolithic architecture is more manageable
		- overhead of managing multiple services isn't worth it
	- API server: handles incoming requests from client and returns appropriate data
	- Database:
		- stores all problems:
			- `id, Description, Code stub, Test Cases, Metadata`
			- can be a NoSQL document like DynamoDB (don't need complex queries and nest test cases as a subdocument)
			- schema:
				- ```
				  {
					  id: string,
					  title: string,
					  question: string,
					  level: string,
					  tags: string[],
					  codeStubs: {
						  python: string, 
						  javascript: string,
						  typescript: string
					  }, 
					  testCases: {
						  type: string,
						  input: JSON,
						  output: JSON
					  }[] // array of input, output pairs
				  }
				  ```
	* viewing a problem: client makes request to API server to w/ `GET /problems/:id` and server returns full statement and code stub
		* use Monaco editor: web-based editor created by Microsoft (also powers VSCode
	* submitting solution: need to be careful
		* bad: don't just run code in API server by saving it as a file, could be malicious, also running code is CPU intensive and could crash server if not managed
			* also no isolation: code crashes, takes down server and no other requests can be processed
		* run code in VM: isolated environment on top of server, can easily be reset if something goes wrong
			* so if user code crashes VM, won't impact server or other users
			* challenges: VMs are resource intensive, slow to startup (need to be careful about running them too long and such)
		* better: run in container:
			* difference versus VMs and containers:
				* VMS run on physical hardware through hypervisor (has entire OS stack, adding overhead but gives strong isolation/security)
				* Containers: share host system kernel and isolate application processes from each other (not full OS, makes them lightweight and does faster start times)
					* less isolation than VMs but more efficient
				* container for each runtime, runs code in sandboxed environment
					* instead of spinning up new VM for each user, reuse the same 
				* challenges: properly configure and secure containers to prevent users from breaking container or use up entire resource
		* best resource:
			* run code in serverless function
				* small, stateless, event-driven in response to trigger
				* cloud providers automatically scale up or down based on demand
				* create a serverless function for each runtime, installs necessary dependencies, runs code in a sandboxed environment
			* challenges:
				* cold start time introduces latency for the first request to the function
				* resource limits impact the performance of long running or resource intensive code (manage them carefully)
	* final submission:
		* API gets code submission and problem ID, sends it to docker container
		* Docker: use `exec()` and container invokes the code and reads output directly from output
		* API server stores result in database and returns result to client
	* view leaderboard:
		* query submission table for all rows based on `competitionID`
		* group by userID, ordering based on distinct problems
			* query:
				* ```sql
				  SELECT userId, COUNT(DISTINCT problemID) as numSolvedProblems, min(submittedAt) as lastSolveTime from submissions 
				  where competitionID == :competitionId and passed = true -- have answered the question correctly
				  group by userID --collapse based on the aggregates
				  order by numSolvedProblems DESC, lastSolveTime ASC --most solved, least solved
				  
				  ```
			* for no sql, create GSI (so make a partition key on competitionID
				* efficiently query without making it the table's primary key (wouldn't work since optional and not unique)
			* would need to be polled continuously
		* user requests through leaderboard endpoint, initiates a query on submission table for database
			* in query or memory, create leaderboard by ranking users by # of problems and ties broken w/ earliest solve time
			* return to leaderboard 
			* client requests leaderboard again to ensure up to date
* deep dives:
	* how to support system isolation or security
		* make the file system read-only and write output to temporary directory deleted short time after completion
		* have strict CPU and memory bounds on the container (if exceeded), gets deleted
		* explicit timeout: if code is running for a while, kill it to fade infinite loops
		* limit network access: don't allow code to make network calls (working within AWS, use VPC Security Groups or NACLS to ban all outbound or inbound traffic)
		* prevent system calls through `seccomp`
	* database be quicker:
		* polling w/ queries: what we currently have
			* not the best approach due to frequent queries + more latency as # of users grows
			* doesn't scale w/ real-time updates (where competitions have many participants)
		* cache w/ periodic updates:
			* introduce redis cache that has the leaderboard
				* gets updated periodically by the database
				* reduces database load
			* issue:
				* updates aren't real-time
				* polling involved (ideal for live updates)
		* redis sorted sets w/ periodic polling:
			* submission processed: database and redis sorted set are updated
			* clients poll server: server returns top N users from Redis sorted set (fast since redis is in memory)
			* processing submission: add to the Redis sorted set the scope and the user ID
			* retrieving top N users, use `ZRange`to get the scores
			* requests the leaderboard: 
				* send them first couple pages of results to display
				* page via client: get next page of results from server, updating UI
			* so flow:
				* submission processed, add to DB, update aggregate score, recompute, update Redis sorted set
				* leaderboard requested:
					* query Redis sorted set w/ range, get top N / requested pages
		* could use websockets, but over kill since we have a 5 second interval
			* can do progressive polling: quicker towards the end, slower at the beginning
	* support larger competitions w/ 100,000 users
		* bad solution: vertically scale
			* not a good solution: there's math that helps w/ this (100 test cases, each test case takes 100ms, submission takes 10 seconds to run worst case)
				* 10000 submissions, 1 million test cases, 100,000 seconds to run (27 hours)
		* great solution:
			* dynamic horizontal scaling
			* have multiple containers for each language distribute submission across them
				* adjusted through autoscaling groups (automatically adjust # of active instances in response to traffic demands, CPU, memory metrics)
			* risk:
				* we can have too many containers, waste resources and get unnecessary costs
		* great solution: horizontally scale w/ queues
			* each container type gets a queue and code gets put onto these queues
				* container pulls submission and ensures we don't lose submissions at peak times
			* worker gets results, notifies server updates both database and cache
			* queue makes this system asynchronous:
				* API server can't return results immeidately
				* need to poll server for results:
					* `GET /check/:id`, polled to check if submission processed
				* websockets: too over complicated, since time is only a second for polling
				* could argue this is overengineering by just handling for peak traffic before competitions
			* queue: allows for requeues of submissions (failiures would be in deadletter queue)
	* handle test cases:
		* need to write a single set of cases for each problem in each language
		* serializes input/output and testharness for each language that deserializes inputs, passes them to users code and compares output to deserialized expected
			* so generic inputs
			* serialize into language-specific struct
			* pass into code
			* compare against expected output (serialization could be done when comparing against custom data structures)