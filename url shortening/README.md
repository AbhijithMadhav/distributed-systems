# URL shortening system

## Functional Requirements
1. Shorten a given URL. How small? As small as possible. Unpredictable.
2. Redirect when presented with the shortened URL. Expiration? 5 years.
3. Delete the mapping
4. Analytics w.r.t. number of clicks of each URL
5. Custom url
6. Serve requests to throughout the world with comparable latencies

## Non-Functional requirements
1. 1 million new requests a day
2. 1:100 creation to redirection ratio
3. Availability via fault tolerance
4. Spike in read requests for popular URLS

##  Data layer
* Items to store
  1. Given url - 1KB
  2. Short url - 20B
  3. Creation timestamp - 5B
  4. Number of clicks - 8B
  5. Expiration timestamp - 5B
* Number of records to be stored in 5 years = 1M * 365 * 5 = 2 billion URLS

Smallest string that can store 2 billion URLS(A-Za-z0-9) = 6 in base 62
Let the shortened url have a 6 char representation

Total storage requirement is 1KB * 2B = 2TB
This will basically fit in a disk => a node

* Shortening url : Write it once. 12 WPS. Spike assumption 120 wps. Can be comfortably handled by a single node
* Redirection : Read multiple times. Read heavy workload. Replication needed.
* 100M read requests per day for an average of 1200 RPS. Assume a spike of 10X. So design for 12000 RPS
* Deletion : Will be pretty sparse. No special consideration
* Counting clicks : Equivalent to redirection. 12000 WPS.

### Consistency requirements
1. A newly generated shortened url might clash with an existing one. So need read committed isolation
2. Concurrently generated urls might clash with one another.
   So need serializable isolation which is stronger than serializable isolation

### Summary for data layer
1. 12000 RPS for redirection - Read replicas required. Maybe caching would do instead. LRU. But having multi-data-center read replicas would help in comparable latencies from different locations. Single leader configuration
2. 120 WPS for url creation - No partitioning required
3. Serializable isolation required. So single leader configuration
4. 12000 WPS for click counts - Partition or batch write since this need not be accurate to the second.
   Use Kafka/kstream(with windowing) for this. Assuming a window of 1 min it will be 200 WPS.
   Assuming a window of 5 min there will be 40WPS bringing the total writes to 160 WPS. No partitioning required

### Conclusion
* A relational db should do with a single leader configuration configured. Say Postgres
* For batching can ingest clicks into Kafka and batch via kstreams
* Caching of shortened URLS

## Endpoints
```
POST /shorten/ with body and custom prefix(optional)
{
"url" : "<long-url>",
"prefix" : "custom-prefix"
}

GET /tinyURL/

DELETE /tinyURL/
```

## URL shortening
* Generate UUID(takes care of randomization) -> Convert it into base62
* Custom url -> No need to do anything



## References
https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/system-design-tinyurl














