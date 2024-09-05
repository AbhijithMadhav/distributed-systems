# URL shortening system

## Functional Requirements
1. Shorten a given URL. 
   * How small? As small as possible and still able to encode a large number of URL's.
2. Redirect when presented with the shortened URL. 
   * Expiration? 5 years.
3. Delete the mapping
4. Analytics w.r.t. number of clicks of each URL
5. Custom url
6. Serve redirection requests with short latencies

## Scale
1. One million new requests a day
2. Creation to redirection ratio is 1:100
3. Availability via fault tolerance
4. Spike in read requests for popular URLS

## Design
![usl-shortening](urls-shortening.drawio.svg)

### URL management service
#### URL shortening
The crux of the design is the encoding/shortening of the url. 
With the expiration of urls specified as 5 years, 
the system must be able to generate non-clashing urls for at least 10 years.
With a million urls to be shortened in a day, 10 years would amount to around 4 billion urls shortened.

Another consideration for the shortened url is readability. 
Using lowercase letters and avoiding special characters will help.
With this in mind, a string of length 7 will be able to encode around 69 billion urls.
Four billion out of 69 is a sparse and gives confidence w.r.t. generation of unique shortened urls.

The process to encode urls is as follows
1. Generate a random number(2468135791013). Using a random number will ensure no dependency on the original url.
2. Encode it to base-26(a-z)
   * 2468135791013 % 26 = 15 = o
   * 468135791013 % 26 = 7 = g
   * 68135791013 % 26 = 17 = q
   * 8135791013 % 26 = 25 = y
   * 135791013 % 26 = 7 = g
   * 35791013 % 26 = 11 = k
   * 5791013 % 26 = 7 = g
3. Encoded string is `ogqygkg`. Shortened url would be `https://<domain>/ogqygkg`

Once this is done, the shortened url can be persisted in a data store. 
However, generated urls might clash with one another.
So need serializable isolation configured for the database.

A relational database configured with serialization isolation will, 
however, lock the entire table during shortening urls.
With the expected WPS of 12(from 1 million shortenings per day) this should work fine though.

A more elegant design would be in ensuring that the encoding generated is guaranteed to be unique. 
The random number generation ensures a low probability of duplication. 
Using a UUID generator would lower these odds more as shortening happens over a cluster of service nodes.

### Deletion and expiration
Deletion of a shortened url is straight-forward.

Expiration can be done by scheduled jobs or by setting a TTL if the underlying datastore allows for the same.
If the former is to be done, partitioning the database by expiration time helps in efficient deletion. 
Entire partitions can be deleted at once.

### URL db
* This can be a relational store.
  * Single leader replication for fail over and for reads due to cache misses
  * Partitioning/Sharding can be done on url so that parallel writes can happen on the db cluster
* Sizing : 1KB * 1M * 365 * 5 = 2TB of data
## Redirection service
There will be a high volume of redirection requests for shortened urls, at about 12K RPS(billion per day).
Reading them of the url db would necessitate reads from replicas to satisfy the request rate.

Since the shortened urls don't change until expiration, 
redirection can be served off a cached server(cluster if required). 
The cache can be populated of a CDC pipeline from the the url db. 
The lag of replication to the cache can be optimized to be low.

On cache miss the lookup can be on the db directly. This should be rare.

### Custom URL generation
This should translate to just another hierarchy in the shortened url. 
If the custom string is foo, the generated url would look like
```text
url: https://www.google.com/larry-page/connon-o-brian/me-me/me
short_url: "https://www.shrt.com/fkuekfg"
custom_url: "https://www.shrt.com/foo/fkuekfg"
```

## Counting clicks: Analytics service
The number of accesses to a URL can be counted by posting count events from the redirection service into a message queue.
These events can then be consumed by a aggregation service to calculate and persist counts in a persistent store. 

Count calculation will typically have more complex requirements like that of a [top-k system](../top-k/README.md).

## To-do
* unique id generator*

## References
* https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/system-design-tinyurl
* https://www.youtube.com/watch?v=5V6Lam8GZo4&t