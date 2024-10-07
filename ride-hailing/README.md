# Raid Hailing(In progress)

## Requirements
1. Users should be matched with cabs
   1. Users specify source and destination


## Ride matching
![ride-hailing](ride-hailing.svg)

## Cab position ingestion
To match nearby cabs with a user location, 
the driver-clients are configured to send current locations every x(say 15 secs).
Since the number of cabs and the rate of request can be high, multiple instances of an ingestion services are required.
Assume a million cabs and a location data from each cab at every 15 secs.
The request rate would be 67K per sec

Since this is a continuous stream of communication, it is conducted over an initially negotiated websocket connection.
No connection setup and teardown is thus required for each ingestion request.

The initial negotiation consists of assigning a client to a particular instance of the websocket based driver-service.
This is done by an assignment service.
The assignment service uses a consistent-hashing-ring-based algorithm to assign clients to a driver service instance.

It keeps track of the status of all the driver-service instances via a coordination service like zk.
At the core, this involves heart-beat mechanisms.

Usage of consistent hashing helps with minimizing migration of assignments
in case an instance goes down or if a new instance is added.
A new instance might also be necessary if any of the instances get overloaded.

In the cases of the handoff described above, care should be taken to prevent the thundering herd phenomenon.
This is typically done by configuring jitter to the clients. 
When assigned to a new driver service instance, they connect with some random variable delay.

The driver service forwards locations to a position ingestion service asynchronously via a queue.
This queue is typically a kafka topic, partitioned by driver-id. 
These messages are [inserted into a storage
backed by a geospatial index](../building-blocks/geospatial-index.md#insertion).

## Ride matching
 
