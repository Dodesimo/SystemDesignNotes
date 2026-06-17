* Functional requirements:
	* Group chats w/ multiple participants
	* Users should send/receive emails
	* Receive/send messages when not online
	* Users should send/receive media in their messages
* nonfunctional:
	* messages delivered w/ low latency
	* guarantee availability
	* handle billions of users
	* centralized servers store messages as long as necessary
	* resilient against failiures of components
* core entities:
	* messages
	* users
	* chats
	* clients 
* API system interface:
	* because there's high-frequency updates sent and received use bidirectional web sockets
	* used over TLS for security 
	* notice how there's no HTTP requests/verbs and stuff
	* `createChat: {"particpants": [], "name": ""} -> {"chatId": ""}
	* `sendMessages: {"chatId": "", "message": "", "attachments": []} -> {"status": "SUCCESS" | "FAILIURE", "messageId": ""}`
	* `createAttachment: {"body": ... , "hash": ""} -> {"attachmentId": ""}`
	* `modifyChatParticpants {"chatId": "", "userId" : "", "operation": "ADD" | "REMOVE"} -> "SUCCESS" | "FAILIURE"`
	* these are commands sent by the server to the client
	* these are all parallel commands sent to other clients 
		* when commands received, send `ack` command (lets us know the command has been received) 
			* this prevents packet loss
	* chat create/update:
		* `{"chatId": "", "participants": []} -> recieved`
	* new message received:
		* `{"chatId" : "", "userId": "", "message": "", "attachments":[]} -> received`
	* these two are commands received by the server from the client
* high level design:
	* start group chats w/ multiple participants
		* L4 (transport layer) load balancer
		* don't use L7 because no need about application information (we don't need to spread requests across many servers based on packet contents))
		* `CLIENT -> L4 LB -> Chat Server -> Database`
			* Database: 
				* `chat: id, name, metadata`
				* `chatparticipant: chatid, participantID`
		* steps:
			* user connects, sends `createChat`
			* creates Chat record in the database along w/ ChatParticipant record for each user for in the chat
				* maps chat id to participant
			* returns chat id to user
		* database:
			* chat table: look up on id, so index/primary key
			* chatparticipant:
				* look up what participants for each chat id
				* and what chat id for each participant ids
					* composite primary key: chatId is partitionKey, participantId is sort key
					* have a global secondary index that keeps participantId as partition key and chatId as sort key
	* send/receive messages:
		* for infra interviews: reason on a single node
		* given single node (obviously horrible for scale)
			* user sends `sendMesssage`
			* chatServer looks at participants based on chatParticipant table
			* look up websockets for each participant in internal hash table, send message through each connection
	* receive messages while not online:
		* keep inbox that contains all undelivered messages
		* if online, send message, else store message, wait to come back later
		* enough write throughput (think about it)
		* sending message:
			* sendMessage to chat server
			* look up all participants
			* write message to table, create entry in Inbox
				* return Success/Failure w/ final message id
			* look up web socket for each participant and deliver message through newMessage
			* for connected clients, client sends ack to Chat Server to indicate message recieved, delete from Inbox
			* if client not connected, when they connect:
				* look up user's Inbox, find delivered message IDs
				* for each message ID look up message in Message table
				* write those to client connection
				* send ack to indicate messages received
			* have a TTL for items in Inbox
	* send receive media:
		* just use blob storage
		* keep attachments in DB:
			* server accept the attachment media over web socket connection save in the database
			* problem:
				* databases can't handle blobs
				* chat servers occupied w/ storage and retrieval
		* send attachment media, to chat server, and push off to blob storage w/ ttl of 30
			* query blob storage through pre-signed URL for authorization
			* expired the media once it had been received by all recipients 
			* disadvantages:
				* chat server: handle incoming media and forward to blob storage
		* manage attatchments seperately:
			* ask chat server for pre-signed URL, file gets uploaded onto it
			* to retireve file, query blob storage direclty through pre-signed URL for auth
* deep dives:
	* billions of users:
		* use more chat servers
			* issue: sending and receiving users might be connected to different hosts 
			* if user A sends message to user B through Server 1 but C connected to Server 2, problem
		* use a Kafka topic per user
			* user connects to Chat Server, subscribe to the topic for User ID, message to given User is published in that topic
			* all chat servers subscribed to that topic would recieve the message
			* issue: not enough topics
		* consistent hashing of chat servers:
			* always assign users to a specific Chat Server based on their user ID (will always know which server responsible for someone)
			* keep central registry of chat servers, addresses, and segments of space
			* request comes in, connect to chat server based on user id
				* new event created, chat servers connect directly w/ chat servers that owns user ID
				* and then calls API that delivers notif
			* challenges:
				* chat servers maintain connections w/ each other 
				* increasing chat servers requires orchestraion of dropping connections to prevent thundering herd of all users moving from one to another
		* offload to pub/sub
			* hashmap of socket connections
			* subscription for given ID then publish messages to that subscription received at most once
			* much lighter than Kafka
			* steps:
				* user connects to chat server, server connects to pub/sub, subscribes to topic for user id
				* any messages for that subscription forwarded to websocket connection for that user
			* needs to be sent:
				* publish message to pub/sub topic for user ID
				* message received by all subscribing Chat Servers
				* forward message to user's websocket connection
			* pub/sub at most once: doesn't guarantee delivery, so message lost
				* its fine because we write to Inbox and Message first
				* light weight compared to other implementation
			* negatives:
				* additional latency (only digit millisecond), connections required between each chat server and each Redis cluster server
	* partitioning by chat or by user:
		* pub/sub topics per user or chat?
			* users have 250 chats each but each chat has another participant
					* 250 channels per connected user
				* partition by user:
					* 1 channel by person
				* here it makes more sense to partition by user
			* users have 1 chat but w/ 100 participants:
				* makes sense to have a channel per chat versus participant
			* so it depends:
				* whatsapp has mostly 1 on 1 chats
			* if the threshold is greater than some number publish on chat else individual
	* clients:
		* track Clients in a table based on user id
		* look up participants for a chat, look up all clients for that user
		* update inbox to be per-client instead of per-user
		* messages get sent to all clients for a user
		* servers keep subscribing to a topic w/ userId
	* connection may fail
		* TCP keep alives don't work because it takes minutes
		* ack timeouts:
			* chat server waits for ACK for client if no ACK arrives, server retries delivery and server assumes web socket is broken 
			* issue: only detect failures when sending messages
		* application-level heartbeats:
			* periodic ping messages over web socket, client responds w/ pong (if no response, server closes connection)
	* redis failiures:
		* what if messages don't get published through pub/sub?
		* periodic polling:
			* clients can periodically poll server for missed messages and get them from Inbox
		* sequence # per chat w/ gap detection
			* each message is monotonically increasing sequence # per chat (generated through )
			* clients track last sequence # seen, if something gets skipped, request a resync
			* challenges: only works if message received, chat goes quiet after missed message, won't detect gap till next message (so need polling)
		* piggybacking on heartbeats:
			* global sequence per user (single incrementing counter per user)
			* sequence added in heartbeat (include user's current sequence number in ping)
			* since this always is sent in increments, don't wait till message received to find a gap
	* out of order messages:
		* can't mitigate
		* users rather want to see new messages as quickly possible instead of ordering
		* chat servers sync time through Network Time Protocol
			* when message is in chat server, stamp w/ time received
			* clients retrieve messages, have timestamp that was recieved by server
			* on clients, display ordered by this time
	* how to do last seen:
		* bad: writing to DB on every heartbeat
			* creates unnecessary load for last seen, struggle w/ load
		* use active connections:
			* take advantage of last disconnect form server
			* when user disconnects, update value w/ current timestamp (conditional expressions to ensure servers aren't racing each other) in a new table
			* special request from client who requests last seen: 
				* ```
				  {
					  //get last seen
					  "targetUserId": "",
					  "requestingUserId": "",
				  }
				  ```
			* update message
				* ```
				  {
					  "targetUserId": "",
					  "reporter": "DATABASE" | "SERVER"
					  "lastSeen": "ONLINE" | "$DATE"
				  }
				  ```
			* workflow:
				* to get last seen, client makes a getLastSeen request
				* server sees this, checks last seen table for targetUserId and does an updatelastseen for the requestingUserId + forward getLastSeen message to Pub/Sub channel for target user ID
				* if chat server receives the getLastSeen and user is connected, updateLastSeen for requestingUserId added
				* then, if client sees ONLINE, bubble is green if not, shows when user last disconnected
					* look into this more later
				* issues:
					* delay between responses for online (waiting a moment)
					* chat servers need to report disconnect times 