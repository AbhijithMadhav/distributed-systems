# Raid Hailing(In progress)

## Requirements
1. Users should be matched with cabs
   1. Users specify destination
2. Cabs nearer to the user should be matched


## Design
![ride-hailing](ride-hailing.svg)

## Cab position ingestion server
* A service which continually ingests cab positions to keep track of cabs. This is useful for two purposes
  * Ride scheduling
  * Ride tracking
* Each cab position, on ingestion, is transformed to a spatial cell(TODO)
* The cab position can be stored in an index which is queryable by cell.
* When 
* Service to ingest locations of cabs(mainly) and users
* Pick a driver
  * Nearest n drivers
  * First response gets selected first
  * factor in rating, fair scheduling