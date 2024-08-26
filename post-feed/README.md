# Post feed and comments

## Requirements
1. User timeline : List of all posts of the user
2. Home timeline : Posting to followers
   1. Users must be able to post
   2. Posts may contain text, image or videos
   3. Users must be able to see see posts of other users they follow, i.e. timeline
   4. Users must be able to see newer posts as they arrive
3. Display followers list
4. Display followed list
5. Comments
   1. Users must be able to comment on other posts
   2. Comments can be nested
6. Search timeline
   1. User must be able to search the contents of posts

    
## User timeline
* Store posts in a table
* User timeline can be rendered by reading off of this table

### Interface

Endpoint: POST /v1/post/{user_id}

Body: Post content

### Posts table
Reads are going to be low compared to the writes. Only a user can view his/her timeline vs a user posting multiple times. Read/Query access pattern will be by user-id and timestamp-sorted.

Overall write by all users will be high. Table will grow large as posts accumulate. So it makes sense to choose a database solution which can handle a high rate of writes. Such databases typically support leaderless or multi-leader writes. A popular example is Cassandra.

Multi-leader or leaderless writes typically imply to write conflicts. In this particular use-case this is not an issue as a write of a post comes from its user and thus cannot come from multiple sources.

To ensure efficient reads,  the primary key should be a composite key of the user-id and tweet-id. This will ensure that the user timeline will result in reads across a few partitions only. This also ensures that the data distribution will be uniform.

If user-id was used as the sole primary key the data distribution would not be uniform.

Timestamp should be used as the clustering key. This will result in the tweets being timestamp-sorted. If the tweets of a user spread across a partition they may have to be merge sorted.

|||
|-|-|
|post_id(primary and partitioning key)|uuid
|user_id|uuid
|media_id|uuid
|post_content|varchar
|post_timestamp(sorting key)|timestamp

![user-timeline](user-timeline.drawio.svg)

## Home timeline
### Interface

Endpoint: GET /v1/home-timeline/{user-id}?cursor={cursor-id}&size={size}



### Naive approach
* Store followers and posts in two relational tables in a single database. To push a post to the followers, join both tables
* Traditional joins in a database are costly, more so when the dataset is large. Posts can be a really big table.
* Needing a join or a secondary index will reduce some database choices that we can make for the posts table or the followers table
* Secondly these join queries will impact primary workflows on these tables by taking away CPU cycles from the db server. Examples
  * A user posting
  * A user following/unfollowing another user
* An argument could be made that we could use replica’s for read(in this case join) load. But a cleaner solution will be to decouple the choice of the database from specific use-case vagaries.
* Using a database to fan out a post to followers will imply implementation of non-trivial application logic for new posts which are posted once a client has rendered out the timeline(Requirement 2d).
* Followers and posts(and maybe even users) would have to be in the same database which might not be desirable

### Streaming approach
* ‘Posts’ table is connected to a stream via CDC. This stream is partitioned by user-id of the post even though the source table is partitioned by a composite key consisting of the user-id and the tweet-id
* ‘Followers’ table(user-id and follower-id mapping) is connected to a stream via CDC. This stream is partitioned by user-id
* Use a streaming platform(Kstreams or Flink) to join these streams by user-id and generate a per follower post.
* The result of the streaming join(per-follower post) can be placed in a caching layer(Redis) by the streaming consumer. This will be a precomputed timeline per user and can be efficiently read off multiple times.
* The streaming joins are efficient(compared to db joins) as they are distributed along partitioning strategy(per user) across the nodes of the streaming platform..
* Updating a new post to a user whose timeline has been rendered will be a matter of pushing the per-user post to clients over the existing web-socket connection. This is done by the same streaming consumer which updates the per-user timeline cache.
* A lot of optimizations could be done to minimize storage at the caching layer also
  * Store references of posts and serve the posts themselves via a cache layer
  * Only the latest posts(say a 1000) could be maintained in the cache. Rare requests for older posts will result in hitting the db table directly.

### Hybrid approach
When a post has a very high fanout(millions of followers), a single new post will result in joins which translates to  writes to a great number of local cache-stores maintained by the streaming platform. This can be slow even with the parallelization that the streaming platform provides.

The expectation is for the post to materialize instantaneously in all the followers feeds/timeline. If this doesn’t happen(within say 5 seconds) users might start seeing reactions to the original post(made by a user with less number of followers) before the original post is rendered.

The solution to this can be reading the posts by such users(with millions of followers) from the database(a cache layer in front of it) rather than updating it to per-user timelines. It would work like the below

* A streaming join for a user with millions of followers should be filtered out.
* When a client asks for the timeline(pull request), a backend service will read and append such posts(of users with high followers) to the timeline(pre-computed by the streaming platform and present in the cache) and then hand the same to the client.
* To update users whose timeline has been rendered(push request), a special message is put into the posts stream.
* The streaming consumer which is responsible for updating the timeline cache will consume this message, read the new post from the database and then push the same to the clients with open web-socket connections

### Followers table
Followers table would see moderate write(following and unfollowing) and read activity. A relational database should do.

With respect to data modeling, the other choice is to map a user with the entire list of followers. But a few users can have millions of followers and this will lead to uneven distribution of data on the storage layer

|                      |      |
|----------------------|------|
| user_id(primary key) | uuid |
| follower_id          | uuid |

![HLD](home-timeline.drawio.svg)

## Comments
Comments have the natural structure of a tree with the root being a post
So comments can be stored in a graph datastore?
But can all comments for a post be stored as a json document in a document store
{
Post : “post-id”,
Comments : [ { comment : “first-comment”, replies : []}, {“2nd-comment”}
}
Posts with lots of comments will result in a big document


## Search




## References
https://www.infoq.com/presentations/Twitter-Timeline-Scalability/
https://www.youtube.com/watch?v=S2y9_XYOZsg&t=946s


## Todo
* Comments
* Search
* Flink checkpointing
* Uuid vs ulid
