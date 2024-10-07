# Tagging system

## Requirements
1. Tag resources(documents, pictures, videos, gifs, tweets etc)
2. Search by tags(more than one possibly. Think of filtered search in LinkedIn) and resources.
3. Statistics based on tags
   1. Most tagged
   2. Most tagged by time interval, region etc
4. Recommend a tag

## Crux
Designing a tagging system is akin to creating an indexing system w.r.t tags.

A relational database with mapping table between tags and resources can be used. 
The table is indexed by tags for a tag search.

| tag-id | resource-id | timestamp |
|--------|-------------|-----------|

For a search by resources, the tags can be stored directly in the resource table. 
The Table here is indexed by resource-id.

| resource-id | resource-url | tags | created | modified |
|-------------|--------------|------|---------|----------|

But does this scale well? 
Assume a resource being tagged with five tags and that the resources can be in the 10's of billions. 
The resource table would have 10 billion entries, and the tag table would have 5X of that.
Both tables would have to be partitioned(sharded) to handle this scale. 
The tag table is partitioned by tag-id and the resource table by resource-id.

For the tag table, this partitioning scheme will create hotspots for heavily used tags. 
Usage of consistent hashing for partitioning could help in splitting partitions as they become hot with popular caches.
Beyond that, caching will have to be used.

What about writes? Once a resource is created and tagged, modifications to the same are going to be minimal.

### Multi-tagged search
Conceptually, this would translate to multiple singular tag search and an intersection of the results. 
At scale(10's of millions of resources) will this be efficient in terms of end user latency?
Manageable as the result set keeps dwindling down with every intersection of the results.

In the worst case, the first search may be for a tag with billions of resources. The response is paged 

### Tag statistics
Tag usage in a specified interval
