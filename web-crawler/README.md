# Web Crawler
## Requirements
* Scrap all content currently accessible via the web
* Store and possibly index all contents
* Respect website crawling policies(Robots.txt files)
* Complete this process in a week

## Crux
* ‘To-Crawl’ list a.k.a Frontier : contains root urls from which crawling is going to start
* Repeat the below steps for each URL
  * Check if the URL has already been crawled
  * Check compliance with hosts robots.txt
  * Get IP address of the host via the root host domain in the URL
  * Download webpage
  * Deduplication : If the contents is the same as the another URL
  * Parse content
  * Store results
  * Add referenced URLS from this webpage into ‘To-Crawl’ list if they were already not visited

## High level architecture
![hld](web-crawler-HLD.drawio.svg)

* URL frontier should be a conceptual FIFO queue from which the crawler service reads the next URL to be scanned and to which it writes referenced URLs in a webpage to be crawled later on.
  * A kind of a BFS approach to scanning the graph of the world wide web.
* The crawler service needs to do the following for every webpage it attempts to download and parse
  * Respect rules of robots.txt w.r.t downloading a webpage
    * Not downloading certain web pages within a host domain.
    * Respecting the ceiling on the rate of downloads. Requeue TODO
  * Download URL only if it is not already not visited
    * This can be done by storing the visited URLS somewhere. Detect already visited references by looking up this storage
  * Check if the contents are already parsed via a different URL
    * Store the content hash of every parsed webpage. Detect duplicate content by comparing content hash of webpage with the stored contents
* Once the page has been parsed and processed by the crawler service
  * The references URLS in a webpage to be crawled later needs to be written
  * The inverted index(word level) needs to be stored in a suitable storage to implement search queries

## Capacity Estimates
* Around 5 billion web pages
* Assume 2 MB for content of each webpage
* Therefore, need to makes 5 billion requests and store 10 PB
* 1 week -> 600K seconds, 5 billion/600K = 7500 requests per second
* Assuming that a webpage can open in around 300ms, would need 2500 executions threads loading the web pages
* Assume that a server has 4 cores we would need around 2500/4 = 625 servers for the crawling service
* Visited URLs cache storage = 5 billion * 200 bytes = 1 TB
* Content hash cache = 5 billion * 64 bytes = 320 GB
* Robots.txt cache : Assume 500 million webpages. Assume 512 bytes per entry. Storage = 500 million * 0.5 KB = 250 MB

## Detailed Design
![detailed](web-crawler-detailed.drawio.svg)

### Crawler service
* Multiple instances(625) of the web crawler are required to complete the crawling in a time bound manner.
* CRUX : Would be beneficial from a caching perspective to partition the url space w.r.t. crawling services. Specifically, partitioning should be by domain host name.
  * This will enable the DNS cache to be also partitioned and local.
  * The visited urls cache could take the partitioning to be by url instead of just hostname. But it can do with hostname just as well to accommodate the DNS cache
* A load balancer picks the next url to be crawled from the frontier and assigns it to a crawler instance based on the hash of the url
* This could be Flink nodes
  * Provides local states which is fault tolerant where all caches can be stored(Checkpointing and all that)
  * If used with Kafka as the frontier partitioning is provided out of box. No need for an explicit load balancer

#### Visited urls cache
* Due to partitioning adopted by the crawler, this will be a local in-memory cache(1TB/625 = 1.6 GB) which stored a subset of urls

#### DNS cache
* Since there is more than one web page per registered domain name, this lookup will help the download much faster
* Due to partitioning adopted by the crawler, this will be a local in-memory cache(250 million/625 = 400K entries * 100 bytes = 4MB) which stored a subset of DNS lookups

### Robots.txt cache
Due to partitioning adopted by the crawler, this will be a local in-memory cache(250MB/625 = 0.4 MB)

### URL Frontier
* One option is for each crawler service to have its own local frontier queue which it discovers while crawling. But this can result in imbalance with one crawler doing a lot of work and others being idle. This situation can occur due the specific seed url with which they start
* Alternative is to have a global FIFO queue which can be accessed by all crawler instances
* Kafka topic suits the bill
  * At Least once message processing
  * Reliability. No lost messages
  * Combined with using Flink for the crawling service partitioning is out of the box. The topic should be partitioned by domain hostname
  * Full replayability in case reprocessing has to be done

### Content cache
* Needs to be a global cache for all instances of the crawler service as duplicate sites can be referred to from two different domains.
* Given the size(320 GB) a single cache server will not be sufficient.
* A Redis cluster with a leader based replication will fit the bill
* Bloom filter  : TODO
  * Since only a not-visited answer is necessary a bloom filter could be used as it is space efficient.
  * If this proves to be something that can be stored within the crawler service, then each instance could maintain its own copy which is replicated from a centralized one. The primary bloom filter is updated by all crawler service instances and the changes are replicated to the local copies of the crawler instances

### Inverted index storage
* Going to be a lot of writes
* So important for storage to be local(same data center) as the crawler instances.
* Storage should accept a lot of parallel writes from the crawler instances
* So need a leaderless or a multi-leader replication. Either an S3 or Cassandra should work
* S3 writes are idempotent. This fits with Flink which can rewrite some stuff on the account if it going down and then doing some reprocessing

## References
* https://www.youtube.com/watch?v=MdWvMX4J-Vc
* https://www.youtube.com/watch?v=0LTXCcVRQi0
* Robots.txt: https://developers.google.com/search/docs/crawling-indexing/robots/intro
* Bloom filters
  * https://krisives.github.io/bloom-calculator/
  * https://stackoverflow.com/questions/4282375/what-is-the-advantage-to-using-bloom-filters
* Search engine
  * https://www.cs.toronto.edu/~muuo/blog/build-yourself-a-mini-search-engine/
  * https://www.baeldung.com/cs/indexing-inverted-index

## Todo
* Bloom filter
* How can the inverted index be stored so that it can be queried
* Top N results
* More info about apache flink
* Searching for words and phrases
