- functional:
	- upload videos
	- watch videos
- nonfunctional:
	- highly available
	- upload/stream large videos
	- low latency streaming
	- scale to high # of videos uploaded/watched
	- resumable uploads
- core entities:
	- User: uploader/viewer or video
	- Video: video that's uploaded or watched
	- VideoMetadata: metadata associated w/ video like uploading user, URL reference
- API:
	- ```POST /upload
	  Request: {video, videometadata}
	  ```
	* stream a video:
		* ```
		  GET /videos/{videoId} -> Video & videometadata
		  ```
* some background info about video streaming:
	* video codec: compresses/decompress digital video making it more efficient for storage/transmission (combined encoder/decoder)
		* trade offs based on time required to compress file, support on different platforms, compression efficiency, compression qualities
	* video container: file format that stores video data (frames, audio) + metadata
	* bitrate: # of bits transmitted over a period of time
		* higher resolution vids w/ > framerates have higher bitrates vs lower resolution videos at lower framerates
		* compression through codecs can impact bitrate (efficient compression leads to a larger video being compressed to much smaller size)
	* manifest files:
		* details about video streams
		* two types: primary and media
		* primary: lists all available versions/formats
			* points to media manifest files (all representing diff. version of video)
			* video version split into small segments (each a second long)
		* media manifest files list out links to clip files and used by video players to stream video as an index
* functional requirements:
	* upload videos
	* where do we store metadata, video data, and what do we store for the video data
	* video metadata:
		* upload rate of 1 million videos/day
		* store in a horizontally-partitioned data base like Cassandra
		* high availability, choose a partition for data
		* partition on videoId, we don't care about bulk-accessing videos just querying individual ones
	* to store data:
		* write directly to blob storage such as S3 w/ a presigned URL that has multi-part upload
	* means that the upload to S3 changes our `POST \upload` to `POST /presigned_url`
	* what do we store video wise:
		* don't store the raw video (native design, doesn't work because all different devices need different video formats for playback)
		* good: store different video formats
			* post-processing of a video to convert the video into different formats playable on diff. devices
			* once user uploads video, S3 fires notif. to video processing service
			* service converts video to different formats (file in s3)
			* update video metadata record
			* issue: need to store small segments of a video for later. this solution doesn't allow partial downloads
		* great: store formats as segments
			* videos get split into small segments (small playable unit)
			* converts each into different formats playable on different devices
			* enables efficient video streaming (we convert small segments into different playable formats)
			* issues:
				* adds complexity
				* post-processing service is more complex w/ a pipeline
				* pipeline splits up segments, generates video formats per segment
				* system needs to store references to segments and then use downstream in streaming flow
	* watch videos:
		* modify `GET /video` to return VideoMetadata, record contains URL to watch videos
		* bad solution:
			* client just downloads whole video and plays it
			* not even streaming just download and playback
			* issue: we need to download the whole video.
				* if entire video needs to be downloaded, user could be waiting a while
				* network disruption: download could fail and we have to restart
		* bettter:
			* we can download segments incrementally
			* client chooses video format based on current device, bandwidth, preferences
			* download first segment, watch quickly without excess loading
			* client could start loading more in the background
			* issue:
				* doesn't take into account network constraints while user is watching video
		* best:
			* adaptive bitrate streaming
			* stored segments of videos in different formats
			* manifest file created that references all video segments available in different formats
				* this is an index for all different video segments in different formats
			* while streaming video:
				* fetch video metadata
				* download manifest file
				* choose a format based on network conditions/user settings
				* client gets the URL from the manifest file, downloads the first segment
				* play the segment and downloading more segments
				* if client detects network conditions are changing, it will vary the format of the video segments its downloading
* deep dives:
	* need to see how video data is processed/stored
	* uploaded in original format, video needs to be post-processed to make available to a wide range of devices
	* output of pipeline:
		* segment files in different formats (codec, container combinations)
		* manifest files stored in s3
			* primary manifest file and several media manifest files
				* primary tells us all different media formats
				* each media format has various segment files for the video
			* media manifest files reference segment files 
	* to generate the segment/manifest files
		* split the original file into segments
		* segments get transcoded (converted to different encodings) and process other aspects of segments
		* create manifest files referencing different segments in different video formats
		* complete
		* an optimization is that its like a pipeline where we split videos and do the process downstream before waiting for the whole video
	* this workflow is a graph
		* there's dependencies
		* segment processing (transcoding, audio processing, transcription) done in parallel
			* no dependencies between segments and the segments have been split up
			* graph of work, this is a DAG
		* expensive computation:
			* video segment transcoding
			* extreme parallelism and on different computers/cores
		* to orchestrate work on this DAG, use a tool ike Temporal that builds graph and assigns worker nodes
			* for tepmorary data in this pipeline, use S3 to avoid passing files
			* URls can also be passed to reference files
	* resumable uploads:
		* client divides video into chunks, each w/ fingerprint hash
		* VideoMetadata has a chunks field (list of JSONS, each w/ fingerprint and status field)
		* client POSTS to backend to update VideoMetadata w/ list of chunks, each w/ status NotUploaded
		* client uploads each chunk to S3
		* S3 acknowledges part upload, returns part number and ETag, client relays through a `PATCH /videos/{id}/chunks`
			* server verifies fingerprint ETAG through S3, update matching chunk record to Uploaded
		* client then calls CompleteMultipartUpload (this is after all items are uploaded), S3 emits object-level notification once per object
			* kicks off downstream processing, chunk-level progress track is client driven tho
		* client stops uploading, resume by fetching video metadata to see what chunks already uploaded, skip chunks already uploaded
	* scale to a lot of videos uploaded/watched a day
		* video service: stateless, responds to HTTP requests for presigned URls and video metadata queries
			* just horizontally scale + load balancer
		* video metadata: Cassandra DB that horizontally scale efficiently (leaderless replication and internal consistent hashing)
			* video ID partition (handle hot shards w/ caching)
		* video processing service: can scale to high # of videos and have internal coordination about DAG distribution
			* service will have internal queueing mechanism that allows for burst handles in video uploads
			* number of jobs in queue trigger system to elastically scale w/ more worker nodes
		* S3: scales w/ high traffic high file volumes
			* bucket lives in a region but has automatic replication across AZs
			* but need cross-region replication or an external CDN for geo-distributed copies
		* hot video: replicate data across multiple shards
			* add a cache as well (distributed, use LRU, partition on videoID)
		* can use CDNs to cache popular video files closer to user to 
			* both segments and manifest files
			* so we never interact w/ backend at all to continue streaming a video
	* also look into speeding up uploads through pipelining
	* resuming streaming 
	* view counts (this is like Youtube Top K type of problem)