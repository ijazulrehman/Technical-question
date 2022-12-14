<!--

-->

# Case

we imagine that we have `100,000,000` offers and that we have `1000` new offers arriving per second. What
do you implement knowing that we want the result of exercise n°1 in near real time?

### Design Requirements and Assumptions

- Our service is used world wide. (Assumption)
- Given a location by a client or user, our service should figure out certain number of nearby locations. (Assumption)
- The infrastructure should be reliable.(Assumption)
- The system should response ≤ 200 milliseconds. So designing for latency is a big concern here. (Assumption)
- 1000 write queries per second.
- Let's assume read requests will be 5x of write requests.
- Each offer will look like this

```
   {
      profession: string ,
      contract_type: string,
      name: sring,
      latitude: float,
      longitude:
    }
```

- As users grow in our system, the system should scale linearly without adding much burden.
- Clients are generally mobile clients, but web clients can also be there. (Assumption)

## Quantitative Analysis

Let's start with `100,000,000` exisiting records.

### Storage

As each offer is represented by `profession`, `contract_type`, `name`, `latitude`, and `longitutde`.

Let's assume some reasonablee numbers for all parameters:

```
   profession = 10 bytes
   contract_type = 1 byte,
   name = 100 bytes,
   latitude = 8 bytes,
   longitude = 8 bytes
```

So to store each record we needs at least: `(10 + 1 + 100 + 8 + 8) = ~127 bytes = ~100 bytes` say.

For 10 million records to start with, we need: `10 * 10⁶ * 100 bytes = 1 Gigabytes`.

With a growth rate of 1000 per secs,

```
    new data per day = 86,400,000 request *  100 bytes = ~ 8.64 Gigabytes = ~ 8 Gigabytes
    new data per month = 8 Gigabytes * 30 days = 240  Gigabytes = ~200 Gigabytes
```

Assuming that each new record will stay for 1 month, and With a growth rate of 25% each year, if we plan for say 5 years, we need to have support for:

```
    1st Year: 200 Gigabytes
    2nd Year: (200 + 200 * .25) = 250 Gigabytes
    3rd Year: (250 + 250 * .25) = ~300 Gigabytes
    4th Year: (300 + 300 * .25) = ~375 Gigabytes
    5th Year: (375 + 375 * .25) = ~450 Gigabytes
```

Just to store the record in our data store, we need a bare minimum store of `450 Gigabytes` remember we may need more storage actually depending on what other metadata / auxiliary information we may need to store depending on our use cases or what metadata our data store internally stores in order to facilitate search or query operation on the data stored. So let’s extrapolate the the storage to a minimum of `500 Gigabytes`.

So how many machines we need to store `500 GB` of data? -> Modern day, there are many database machine types (in AWS or Azure) which can offer storage from `16 TB to 64 TB` depending on cost & use cases. So ideally 1 machine should be enough to support our use case. But remember, with a higher growth rate or with increasing popularity, we may need more storage or more machines. Nevertheless, we will design our infrastructure to support horizontal scaling.

### Read Heavy vs Write Heavy System?

Our requirement already specifies that we have assumed that read is `5x` of write. So it’s already established that our system is extremely read heavy & the Read:Write ratio is `5:1`.

### Network Bandwidth Estimation

With 5000 read and 1000 write requests, Each read request will contains 10 records (pagesize = 10).

- `content-lenth` of read request `(10 records * 100 bytes) + ~800 bytes request meta = ~ 1800 bytes `
- `content-lenth` of write request `(1 records * 100 bytes) + ~800 bytes request meta = ~ 900 bytes `

```
data transfer/sec = (5000 read requests * 1800) + (1000 read requests * 900) = 9.9 Megabyte = ~10 Megabyte
```

### Network Protocol Choice

we have assumed to support web & mobile clients, we need to choose a protocol which is developer friendly, easy to use & understand as 3rd party developers need to integrate with our infrastructure. There are couple of options like — SOAP, RESTful API over HTTP or RPC API.

- **SOAP**: It’s quite old & complex to understand, pretty nested schema & typically XML is used to express the request & response. It’s not very developer friendly as well. We won’t choose SOAP.

- **RPC**: It’s good for use cases suitable for communication in the same data centre & RPC requires client & server side stubs to be created which makes them tightly connected. Changing the client-server integration will be painful in case there is any need. So we won’t choose it.

- **RESTful with HTTP**: This is something which nowadays has become the de-facto standard. It’s developer friendly, mostly developers use JSON schema to express request / response which is well understood in the community, easy to integrate & introduce new API version when required. For these reasons, we will go ahead with designing our API as RESTful API using HTTP.

### Database Schema

Since our data is static, it makes sense to put data into a data store since these locations are mostly static. On other hand If it will be very dynamic like cab’s location or delivery boy’s location & those locations constantly get pushed to our system, it does not make sense to store them in persistent data store unless we want to enable tracking for this or that purpose.

### High Level Design (HLD)

<p align="center">
  <img src="./images/hdl-1.png" label="High Level Design">
  <h4 align="center">High Level Design<h4>
</p>

#### Load balancer

**1. Why do we need a load balancer here?**
Considering in the worst case all the traffic come from the same region, it will be difficult for a single server to manage that load. So we need stateless application servers which can share the load among themselves. Hence the load balancer comes into action.

Now our problem statement becomes:
**How to make the locations searchable since our data store might not natively supports it or even if it supports, we can’t put so much of load with growing scale?**

### Approach: Create partitions in memory & search in-memory

Our initial data size is around `2 Gigabytes` & it can grow up to `450 Gigabytes` or more in near future. Since we need to support massive read queries, we can do aggressive caching, but depending on the cost, we can start by using single machine, but with time we need to have distributed cache to accomdate hundreds of Gigabytes.

_What shall we store in the cache?_
In this approach, for any location query, we are finding distance between the client provided (lat, long) & all locations stored across all cache machines. After finding out all the result, we rank the locations based on ascending distance order & client provided criteria if any. This is basically a brute force search & we do it completely in memory, so it’s faster than doing the whole operation in database provided our database does not support location queries. It does not matter in what format we store the locations, we may store the location id as key & the location object as value or if the cache supports list of objects, we can store list of locations against any key & search across the list of locations.

_How will the cache get loaded?_
We can have a scheduled job which will run every few minutes and load the locations since the last id or timestamp. This is not a real time process, so we can have our write path also write / update the location information when the POST location API gets called. In the following architecture the write path has 2 steps:

- Step 1: Write to the database,
- Step 2: Write to the cache.

_What is the strategy to partition the cache machines?_
There are many strategies to partition the cache machines, Here in our case locatin based sharding will be more approriate. In order to manage these shards & balance the load evenly across all possible cache servers, we will use a central metadata registry. The metadata registry awill contain mapping from region to logical paritition id, logical shard id to in-memory index server id mapping ( using service discovery mechanism. Both the read & write request first talk to the metadata registry, using the IP address of the request’s origin, we determine in which reagion the delivery agent exactly is or from where the user request came. Using the resolved region, shard id is identified, then using shard id, index server is identified. Finally the request is directed towards that particular index server. So we don’t need to query all the cache / index server any more. Note that, in this architecture, neither read nor write requests talk to the data store directly, this reduces the API latency even further.

<p align="center">
  <img src="./images/hdl-2.1.png" label="High Level Design">
  <h4 align="center">High Level Design<h4>
</p>
