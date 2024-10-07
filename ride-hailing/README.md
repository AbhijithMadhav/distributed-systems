# Ride Hailing

## Requirements
- Users should be matched with cabs


## Ride matching
![ride-hailing](ride-hailing.svg)

## Cab position ingestion
To match nearby cabs with a user location, 
the driver-clients are configured to send current locations every x(say 15 secs).
Since the number of cabs and the rate of request can be high, multiple instances of an ingestion services are required.
Assume a million cabs and location data from each cab at every 15 secs.
The request rate would be 67K per sec

### Websocket connection : Driver service
Since this is a continuous stream of communication, it is conducted over an initially negotiated websocket connection.
No connection setup and teardown is thus required for each ingestion request. 
Further communication during ride matching negotiation and the ride itself can be done over the same connection.

### Geospatial index
All the location data would have to be stored in a [geospatial index](../building-blocks/geospatial-index.md) 
for later ride matching. 
Densely populated areas will have more cabs and hence a lot more location requests.
The index will have to be partitioned according to this distribution pattern to ensure uniformity in load handling.

The partitioning will be by geo-hash range with denser areas mapped to partitions with a smaller geo-hash ranges.
For ride matching, all the required nearby cabs will typically be found within the same index partitioning.

### Assignment service for load balancing
Each partition of the geospatial index is fronted by a websocket based position ingestion service.
A load balancer service does the initial assigning a client to a particular instance of the ingestion service.
The assignment service uses a consistent-hashing-ring-based algorithm to assign clients to a ingestion service instance.

It keeps track of the status of all the ingestion-service instances via a coordination service like zk.
At the core, this involves heart-beat mechanisms.

Usage of consistent hashing helps with minimizing migration of assignments
in case an instance goes down or if a new instance is added.
A new instance might also be necessary if any of the instances get overloaded.

In the cases of the handoff described above, care should be taken to prevent the thundering herd phenomenon.
This is typically done by configuring jitter to the clients. 
When assigned to a new ingestion service instance, they connect with some random variable delay.

### Cab handoff on grid crossover
From time to time a cab will cross over from one geo-hash range to another. 
This translates to change in assignment of the cab to the ingestion service instance. 
This happens by the current ingestion instance rejecting the location ingestion request and closing the connection.
This forces the client to ask for a fresh assignment with the assignment service which then assigns it to a new 
ingestion service instance based on its location request.

### Storing location data
The location information can be maintained by the individual ingestion service instances in memory.
Any loss of the node results in a standby node taking over and the clients reconnecting. 
The location information is re-sent by the clients after reconnecting. 
Not persisting this info in storage helps with efficiency too.

## Ride matching

### Rider service
A websocket based service interfacing between the rider and the system. 
Similar to the driver service. Handles
- Rider location ingestion
- Ride request
- Ride matching negotiation
- Possible ride tracking(not in scope)

### Ride matching service
For each ride request,
- Get list of suitable cabs based on
  - Nearby
  - Rating
  - Cab with less number of trips
  - etc
- Create a trip
  - Create an unclaimed trip entry in a relational db
- Send ride offers to shortlisted cabs
- Process ride offer response from cabs and shortlist one
- Mark trip as claimed
- Send claimed trip details to rider and cab


## TODO
- Ride requests on the periphery of the geo-hash range will require data from partitions hosting neighboring geo-hash ranges.

 
