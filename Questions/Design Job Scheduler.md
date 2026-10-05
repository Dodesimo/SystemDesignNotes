- task: abstract concept of work that's done
- job: instance of a task, made up of task, schedule for task, parameters needed to execute job
- core requirements:
	- function
		- schedule jobs immediately, future date, recurring schedule
		- monitor status of jobs
	- nonfunctional:
		- highly available
		- execute jobs within 2 seconds of scheduled time
		- scalable to support 10k jobs per second
		- at least once: keep executing till it runs
- infra:
	- core entities, api, data flow, high level design, deep dives
- core entities:
	- task: task to be executed
	- job: instance of task to be executed at given time w/ set of params
	- schedule: when job should be executed (CRON expression or specific DateTime)
	- user: person who schedules job and views status of job
- API:
	- ```
	  POST /jobs {
		  "task_id": "send_email",
		  "schedule": "0 10 * * *",
		  "parameters": {
			  "to": "john@example.com",
			  "subject": "daily report"
		  }
	  }
	  ```
		* this creates a job
	* querying status of jobs:
		* ```
		  GET /jobs?user_id={user_id}
&status={status}&start_time={start_time}&end_time={end_time} -> Job[] 
		  ```
* data flow:
	* what happens as request enters system to when it produces final output
	* so:
		* user schedules job by providing task, schedule for when task executed, params needed to execute task
		* job is persisted in system
		* job picked up by worker, executed at scheduled time
			* job fails, retry w/ exponential backoff
		* update job status in system
* high level design:
	* schedule jobs to be executed immediately at a future date or on recurring schedule
		* request to `/jobs` w/ task id, schedule, parameters
		* store job in DB w/ `PENDING` status
			* persist record of jobs
			* recover jobs if system crashes
			* track job status throughout life
			* choosing DB: doesn't really because there's no need for strong consistency so can just use a key value store like DynamoDB
		* this approach doesn't work for recurring schedules
			* store the CRON expression in table, but how do we effectively find jobs that run in the next few minutes
		* separate definition of job from execution instances
		* so split data into two tables:
			* Jobs table has job definitions:
				* ```
				  {
					 "job_id": "123e4567-e89b-12d3-a456-426614174000",  // Partition key for easy lookup by job_id
					  "user_id": "user_123", 
					  "task_id": "send_email",
					  "schedule": {
					    "type": "CRON" | "DATE" 
					    "expression": "0 10 * * *"  // Every day at 10:00 AM for CRON, specific date for DATE
					  },
					  "parameters": {
					    "to": "john@example.com",
					    "subject": "Daily Report"
					  }
				  }
				  ```
			* execution tables has each individual time a job should run
				* ```
				  {
					  "time_bucket": 1715547600 (Partition Key of Unix timestamp rounded down)
					  "execution_time": "1715548800-123e4567-e89b-12d3-a456-426614174000" (sort key, exact exec time + job Id to ensure unique primary key)
					   "job_id": "123e4567-e89b-12d3-a456-426614174000",
					   "user_id": "user_123",
					   "status": "PENDING",
					   "attempt": 0
				  }
				  ```
				* time bucket of Unix timestamp rounded down to the nearest hour, we can efficiently query for upcoming jobs
					* query current hours and next hour's bucket
				* calculated w/ `time_bucket = (execution_time // 3600) * 3600`
				* recurring job completes, schedule next occurrence by evaluating next execution time and creating new entry in `Executions` table
				* worker node ready to execute jobs, query table for entries where `execution_time` is within the next few minutes, pending status
	* monitoring of status:
		* job executes, update the table w/ COMPLETED, FAILED, IN_PROGRESS, RETRYING enum values
		* how to find status of all jobs for a given user:
			* right now, query Jobs table for all job_ids for user 
			* then query Executions Table to find status of each Job
		* better:
			* add a GSI on Executions table that has a partition key of `user_id` and sort key of `exceution_time` and `job_id`
				* allows us to find all executions for a user and sort they by the time and job_id
			* GSI allows for multiple access patterns instead of data denormalization
* deep dives:
	* ensure system executes job within 2 seconds of scheduled time?
		* querying DB every few minutes to find jobs due for execution
		* frequency w/ how often we run cron is upper bound of how often we execute jobs
		* this means we would have to run the CRON every two seconds
		* doesn't work:
			* each query needs to fetch/process around 20k jobs (10k jobs per second, we want jobs due in the next two seconds), large payload that is hard to process
			* querying about 20k jobs introduces a lot of latency
			* initialize job, distributing them to workers, and beginning execution adds latency
			* running large queries every two seconds add load to the DB
		* solution:
			* add two-layered scheduler:
				* query the database (Execution table) due for next five mins
				* add the jobs returned to a message queue that's ordered by `execution_time`
					* so that means we work on the jobs that are the in the queue most recent
		* why is this better:
			* reduce load on the database
			* message queue has high throughput, workers can pull and process jobs as fast as they are available
		* what about new jobs that are created and expected to run in less than five mins
			* they would miss in the five mins
		* can add directly to message queue
		* issue: Kafka is partition based so new job goes to the end of the partition and waits behind all jobs that are already queued, even if scheduled to run sooner
		* queue system needs to support delayed delivery: jobs become visible to workers at or near scheduled execution time
		* good solution:
			* redis sorted sets:
				* use execution timestamp for ordering: job needs to be scheduled, add to sorted set w/ execution time as the score
					* log n insertion
				* efficiently query for the next jobs by fetching entires w/ scores less than current timestamp
					*  since these have to execute already
				* could be better:
					* implement retry logic, handle failures, manage replication
			* rabbit mq:
				* use TTL + dead letter exchange pattern
				* publish messages to queue w/ per-message TTL, configure dead-letter exchange that routes expired message to processing queue
				* handles message persistence and publisher confirms for reliable delivery
				* issues:
					* lots of things w/ quorum queues and clustering
					* adds complexity compared to native delay support
		* best:
			* amazon SQS:
				* fully managed queue service that has delayed message delivery
				* scheduling a job, send message to SQS w/ delay value
				* schedule a job to run in ten seconds, send message to SQS w/ delay of ten seconds
				* delivery delay feature ensures messages are invisible till scheduled execution time
				* dead-letter queues capture failed jobs for investigation
				* we have availability zones and autoscaling
		* DelaySeconds is a minimum delay, not a precision guarantee but since workers are continuously polling we don't have additional latency
		* flow now:
			* user creates new job, write to DB
			* cron runs every 5 mins, queries DB For jobs that need to be executed in five mins
			* cron sends to SQS w/ delay values
			* workers poll SQS and process messages as visible
			* if new job made with scheduled time < 5, send to SQS w/ its delay
	* how to be scalable to support 10k jobs:
		* job creation:
			* ask what the distribution of creation is (per second versus one time)
			* if most jobs are one-time job creation might be a bottleneck
			* add a message queue between API and Job Creation service, queue is a buffer during spikes
				* allows job creation to process requests at a sustainable rate 
				* message queue allows horizontal scaling by adding more consumers while giving us durability
			* this is likely overcomplication, DB should be able to handle write throughput directly, always have a service in front that can scale horizontally
		* jobs db:
			* partition keys allow us to spread across many partitions
			* jobs table: partitioned by `job_id`, distributes writes evenly
			* executions table: partitioned by time_bucket, need to be thoughtful since all writes for the current hour land in the same partition (creates hot shard problem)
				* write sharding w/ a random suffix to the partition key, which is then done by all workers to query all shards for a given time in parallel
			* can also move job to cheaper storage solution after time has passed
		* workers:
			* should be containers or lambda functions
				* containers: more cost-effective, better suited for long running jobs since they maintain state (more overhead, don't scale as elastically)
				* lambdas: server-less, with minimal operational overhead (great for short-lived jobs) under fifteen minutes, can auto-scale to match workload
					* there's a cold start
				* use containers w/ ECS and auto-scaling groups, optimize setup w/ spot instances, auto-scaling based on many items in queue, prewarming container pool, etc.
	* ensure at least once execution of jobs:
		* jobs can fail based on visible failures (bug in task code or incorrect input parameters)
		* invisible failure: worker itself went down
		* visible:
			* try/catch in code to log error and mark failed
			* retry w/ exponential backoff
			* put jobs back into the message queue w/ increasing delay based on retry:
				* first retry waits 5 seconds
				* second waits 25 seconds
				* third waits 125 seconds
				* do this by setting DelaySeconds parameter when re-enqueuing message
			* if a job retries 3 times and fails, mark status as FAILED and job no longer retries
			* SQS tells us ApproximateReceiveCount and dead letter queues catch messages that exceed retry limit
		* invisible failures:
			* each worker exposes a `GET /health`, centralized monitoring service polls endpoints at regular intervals, when worker fails to respond to multiple checks, mark it as failed and reassign it to healthy workers
			* don't scale w/ thousands of workers
			* network issues b/w monitoring service and workers can trigger positives
			* lots of additional infra, what if the monitoring service goes down itself
		* job leasing:
			* distributed locking mechanism using a database to track job ownership
			* worker wants to process job, acquires a lease by updating job record w/ worker ID and expiration timestamp
			* while processing job, worker periodically extends lease by updating the expiration timestamp
			* if the worker crashes, fails to renew lease, expiration time passes, another worker acquires the lease and retries the job
			* challenges:
				* process frequent lease renewal operations add DB load: 10k jobs per second, 50k lease updates per second
				* clock sync: Worker A's clock says 10:30:00 but Worker B's clock says 10:30:20, Worker B steals job while Worker A is processing
				* leads to duplicate execution
		* SQS visibility timeout:
			* worker gets a message from queue, SQS makes it invisible to other workers for a period
			* worker processes message and deletes upon completion
			* worker crashes/fails within the visibility timeout period, SQS makes message visible again for others to process
			* set a short visibility timeout and have workers periodically heartbeat through ChangeMessageVisibility
				* worker processing job could extend visibility every 15 seconds
				* worker crashes, other worker picks up job in 30 seconds instead of a longer timeout expiring
			* gives us fast failure detection without additional infra or complex coordination
		* need idempotency guarantees:
			* deduplication table:
				* have a table that stores successful job executions w/ job id and execution timestamp
				* adds DB operations every job execution + requires maintaining another table
				* the other table needs to be cleaned up periodically as well to prevent unbounded growth
			* idempotent job design:
				* make the jobs have idempotency keys and conditional operations
					* not just increment counter, but set count to X value
					* for "send welcome email," check if welcome email flag is set in profile
				* each job execution has a unique ID used for deduplication
				* 