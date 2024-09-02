# Top K leaderboard

Finding the top k popular items is a use-case seen in applications. Popularity is measured by views, likes, references, etc
1. Top k trending tweets
2. To rank search results
3. Most cited paper
4. Most viewed/liked videos etc

## Requirements
1. Find the top k events by count.
2. Top k should be returned for any query interval.
3. YouTube has around five billion views in a day(as of 2024). 
   Keeping views is a count metric, the system should be able to handle several billions of count events. 
   Say 10 billion. This translates to 100k events per sec
4. There are around nine billion google searches in a day. So the system should answer a top-k query rate of ten billion 
   a day. Translates to 100k queries per sec.

## Count events ingestion
The ingestion rate is very high(100k events per sec).
This rules out any naive, single node, in-memory hashmap based counting.
* A single service host can't cope with the ingestion rate
* A single service host can't store counts for all the events. 
  Consider the space required for the hashmap in the following cases, 
  assuming each entry in the hashmap requires 20 bytes
  * Top ranked websites Search use case: 50 billion websites ⇒ 1 TB
  * Top viewed/liked videos on YouTube: 5 billion of total videos ⇒ 100 GB 
  * Top k tweets : 500 billion tweets per year ⇒ 10 TB 


A service cluster is required to receive the count requests. 
They will then push the same to a distributed messaging queue for asynchronous processing. 
Event-id partitions the queue. 
This ensures a good near-uniform distribution of events across partitions of the queue. 
This also helps with parallelism in consumption.


## Trade-off: Accuracy vs real time response
It is a given that any top-k query will not reflect the absolute latest count event like a transactional system. 
Instead, the goal of the system is to be as near realtime as possible.

For the query result to be accurate to the last count, batch processing as per the query interval is required.
This is a slow process and can't be near realtime. 
So the system can offer two SLAs
1. Accurate but slow
2. Fast with approximations

The former can be used for backend processes and analysis. 
The later can be used for use cases outlined in the requirements which have a outside user interaction.

## Accurate path
Every count event is dumped into a distributed file system(DFS), say HDFS. 
Some batching in the queue consumer is done before writing to the DFS(TODO: Why?).
In response to a query request, a map-reduce job is spawned which returns the top-k result. 
This obviously takes some time as the result is calculated from scratch.


![accurate-topk](accurate-batch-processing-based-topk.svg)

### top-k calculation using map-reduce
1. Count phase
   1. At each DFS node, for the query interval, calculate the count by event-id
   2. Obtain global counts per event via a shuffle of data across nodes
2. Top-k
   1. Use a k-sized min-heap to find out the top k 
   2. Use of k-sized min heap technique is O(nlogk) versus the O(nlogn) if sorting or max heap approach is used. 
      With k being considerably smaller than n, the min-heap approach is efficient.


![map-reduce diagram](shuffle-min-k.svg)

## Fast path with approximations
To answer queries quickly, some preprocessing of the count events is required. 
Preprocessing is possible by applying constraints on the query intervals that can be made for top-k. 
Let's say that any query must start and end at a clean hour.
It can span several hours or days as long as the boundaries are clean as specified before.

![approximate-realtime-approach](approximate-stream-processing-based-topk.svg)

The count events in the topic are consumed by a streaming application which calculates the hourly top-k results.
The streaming application consists of two main phases. 

#### Windowed aggregation: Count of events in every hour
  * The crux counting of events in every hour hopping windowed aggregation
  * Hopping windows of one-hour lengths are configured
  * Counts of events using simple hashmaps are done within the window duration
  * Since event-id partitions the input queue, all events with the same event-id within an hour are redirected to the same 
    host/thread in the streaming application
  * Thus, when the windows close, we have counts of all events within that hour. 
  * Note that these event counts are distributed over the nodes of the streaming application and have to be brought 
    together to figure out top-k

#### Reduce: Hourly top-k calculation
  * Once the window is closed, the counts aggregated within the hour are forwarded to a topic
  * A consumer application consisting of a single consumer takes the hourly count of all events from all streaming 
    nodes and calculates hour-specific top-k's using a k-sized min heap
    * Single consumer because this task can't be parallelized
    * Fault tolerance has to be ensured by the orchestration framework
    * Due to delays in processing, it is possible for hourly counts from more than a single hourly interval to be 
      present in the queue.
    * The consumer will make use of possibly multiple min-heaps in this scenario
  * This top-k is pushed into storage, typically a time-series db(WHY???). 
  * The consumer will do this at regular intervals as the end of the event stream for a particular hour 
    can't be known in advance
  * Note: The top (k + 1)th entry will be lost. The implication of this is inaccuracy in the calculation of the global top-k.
  * So every hour the time-series db has k rows written to it.

### Top-k query processing
When a top-k query is done 
* Fetch all rows for the interval specified from storage
* At the application layer calculate top-k using the k-sized min-heap approach and return the result
* There is an obvious inaccuracy in the result, which is a tradeoff for the realtime response
  * Consider an event which did not qualify for the local top-k in the earlier phase, say, the (k + 1)th event.
  * It may have had a higher aggregated count across the several one-hour intervals specified by this query.
  * But it is not a candidate to calculate the global top-k.

### Count-min-sketch
With a streaming approach, in some cases, memory consumption can be high while accumulating the 
counts in the windowed aggregation. 
A count-min-sketch map can be used instead.
This is a probabilistic data structure and has a constant memory footprint irrespective of the number of key-value 
entries that can be stored.
The tradeoff here is occasional over counting.

## References
* https://www.youtube.com/watch?v=nQpkRONzEQI
* https://www.youtube.com/watch?v=kx-XDoPjoHw

## TODO 
* Why time-series database? What is special about it?
* Ingestion into HDFS. Why is there a need to shuffle? Can't events with the same id be inserted into the same node??


