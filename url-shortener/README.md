# Notes from this [article](https://systemdesign.one/url-shortening-system-design/)

## Url Shortener API

### Url Shortening

- The client that wants to shorten a url executes an HTTP PUT to the shortener server. The header contains authorization token, browser used (user-agent), and some other fields. The body of the request contains the long url, and optionally: tags, expiration period, and the short url itself.
- The response to this request from the server can be:
  1. 200 OK when successful. The header contains some data including type of data format, the body includes the long and short urls, the creation date, and whether the url is activated.
  2. 401 Unauthorized if the client didn't provide the credentials or didn't have valid credentials.
  3. 403 Forbidden if the client has valid credentials but not sufficient privileges to act on the resource.

### Url Redirection

- The client can now execute an HTTP GET request to the shortener server to get the short url. There's only a header than contains auth, cookie, user-agent, but not the short url. idk how the server gets the short url at this point. maybe in the cookies, or maybe this request is made immediately after the last request?
- The response to this request can be:
  1. 301 Moved Permanently to indicate the short url is now permanently moved to the long url. The long url is supplied in the header as the value of the "location" key.
  2. 302 Found, to indicate the resource has been temporarily moved to a new location, with the location being supplied in the header.
  3. 307 Temporary Redirect, which somehow makes the client execute a PUT request to the original long url using the same body of the first request.
  4. 404 Not Found if the short url doesn't exist in the database.

## Database Schema

- The database consists of two tables: Users and URLs. the relationship is 1-to-many. The Users table has an id, name, email, created at and last login columns. The URLs table has the short url as the primary key, the long url, created at, expires at, last_visited, and user id as a foreign key.
- A NoSQL db is used to store the URL table, a SQL is used to store the Users table.

## Capacity Planning

- If a url is approximately 2.5KB is size, then the size required for a 5 year url storage, given there's 100 million urls added per day is 2.5 KB _ 100 million _ 300 days (for simplification?) \* 5 years = 450 TB required storage. We double this by 3 for improved durability and disaster recovery, which makes the storage needed 1.35 PB.
- The ingress (client requests, network traffic entering the server) is 2.5KB _ 100 million urls per day _ 10^-5 for simplification to make it per second instead of per day, results in 2.5 MB/s required bandwidth for ingress.
- The Egress (server responses, network traffic that exits the server) is 2.5 KB _ 10 billion _ 10^-5 = 250 MB/s.
- This egress is cached to improve the latency. Approximately 80% of the egress is served using only 20% of url cached data. The remaining 20% egress is grabbed from the non-cached portion. We cache used urls for a day. Since our egress is 250 MB/s, which is 25 TB/day roughly, we cache only 20% of those, which makes our memory requirement 5 TB/day.

## High-Level Design

### Encoding

- We use base 62 which ranges all characters and numbers. Each character consumes 6 bits. a url of 7 characters consumes 42 bits. base 62 is made of 26 small letters + 26 capital letters and 10 digits = 62. since we decided to use 7 characters, 62 choices with repetition is 62^7 = 3.5 trillion possible unique ids. 3,5 trillion is exhausted in 100 years when 1000 short urls are generated per second. The time complexity of base conversion (generating a short url?) is O(1).

### Write Path

- We move out the Key Generation Service (KGS) to scale out the system. We can use one of three solutions to shorten a url: 1. Random ID Generator. 2. Hashing Function. 3. Token Range.

#### Random ID Generator Solution

- The ingress is distributed using a load balancer, with algorithms like random, round robin, etc. The load balancer then routes the traffic to one of multiple random id generators. The random id generators use a random function or a Universally Unique Identifiers (UUID).
- The random id generation solution must verify with the DB every time whether the short url has been used before.
- An alternative to this is Twitter's Snowflake, which has a 64 bit output length. This however also has overlaps, which means we also mush check the DB first.
- The Random ID Generator is _not_ recommended as a url shortening solution.

#### Hashing Function Solution

- We apply the same load balancer concept here as well. We can use MD5 hashing function which outputs 128 bits. Or SHA256 of length 256 bits.
  1. MD5: We take the first 7 characters from the 128/6 (bits used per base62 encoded characters) = 22 characters. After which we append random bits to the suffix to make the output nonpredictable, at the expense of readability.
  2. SHA256: The probability of collision is higher than MD5 because we also take the first 7 characters from the 256 bits.
- The Hashing Function is _not_ recommended as a url shortening solution.

#### Token Range Solution

- The load balancer uses consistent hashing to distribute.
- The output of the token service must be non-overlapping.
- The token range service is a highly reliable distributed service such as Apache Zookeeper which is used to coordinate the output range of token service instances. This service might become a bottleneck. We solve this be either increasing the replicas or the output range.
- We must check the existence of the long url in the db first before generating a short url. We use a bloom filter to prevent expensive data store lookups.
- We introduce an additional data store with inverted index mapping long urls to short urls instead of the opposite, to solve the problem of finding a short url using the long url.
- The time complexity is O(1).

### Read Path

- We also cache following the 80/20 rule to improve latency.
- When a cache miss occurs, we query the data store to populate the cache. We use Least Recently Used (LRU) policy to evict the cache when the server is full.
- Since the typical usage pattern of a url shortener is to access a short url once, this results in cache thrashing. To prevent this we use bloom filters, which are updated on initial accesses to short urls. On subsequent accesses, the server checks the bloom filter first. If the short url doesn't exist, it returns a 404. If it exists (which might be a false positive), the cache server is checked, then if needed, the data store.
- We can replicate the cache servers and use one for write operations (leader cache), and the rest for read operations (follower caches).
- If multiple identical requests arrive to the cache server, the requests will be collapsed, and the response reused among all clients.

## Design Deep Dive

### Availability

- Availability can be improved by:
  - load balancer runs either in active-active or active-passive mode.
  - KGS runs either in active-active or active-passive mode.
  - back up the storage servers at least once a day to object storage such as AWS S3 to aid disaster recovery.
  - rate-limiting the traffic to prevent DDoS attacks and malicious users.

### Rate Limiting

- We use API keys for registered clients, and cookies for anonymous clients to be able to identify malicious clients and rate limit them. IP is also used to rate limit.

### Scalability

- We do the following to scale a system:
  1. benchmark or load test.
  2. profile for bottlenecks or a single point of failure (SPOF).
  3. address bottlenecks.

### Fault Tolerance

- We use a microservice architecture to improve fault tolerance; if one component fails, it fails alone.
- We use a message queue such as Apache Kafka or RabbitMQ to isolate components, allow concurrent operations, make it fail independently, and asynchronous processing, at the cost of system complexity.

### Partitioning

- We partition these services using partition keys of either the long url or the short url.
  - bloom filter in write path uses long url partition key.
  - bloom filter in read path uses short url partition key.
  - data store and cache use short url partition key.
  - data store (inverted index?) uses long url partition key.

### Concurrency

- The Key Generation Service (KGS) must acquire a mutex lock or a semaphore on the atomic data structure distributing the short urls to handle concurrency. This lock prevents distributing the same short url to multiple concurrent long url requests.
- We can use a message queue, collapsed forwarding feature of a reverse proxy, or a distributed lock service to solve the problem of clients entering the same long url at the same time must receive the same short url.

### Analytics

- The HTTP headers of url redirection requests are used to collect data for the generation of analytics. The most popular HTTP headers useful for analytics are: user-agent, cookie, authorization, referer, and date.

### Database Cleanup

- To remove expired records in the database we use either:
  1- Lazy removal: when the client accesses an expired url, remove it and respond with 404.
  2- Dedicated cleanup service: executed during non-peak hours to remove expired urls by scanning the whole database. If the traffic spikes, the service mush be stopped, and the system would fallback to lazy removal.

### Archive

- he data that is frequently accessed is classified as hot data and the data that was not accessed for a significant amount of time (assume a time frame of 3 years) is classified as cold data.
- The last_visited timestamp column of the database is used for data classification. The cold data is compressed and stored in object storage (AWS S3) during non-peak hours to avoid degradation of the service.
- The unused data must be archived only if you are certain that the data will not be accessed in the future.

### Monitoring

- Monitoring is usually implemented by installing an agent on each of the servers (services). The agent collects and aggregates the metrics and publishes the result data to a central monitoring service.

### Security

- The following list covers some of the most popular security measures:
  - use JWT token for authorization.
  - rate limit the requests.
  - encrypt the data.
  - sanitize user input to prevent Cross Site Scripting (XSS).
  - use parameterized queries to prevent SQL injection.
  - use the principle of least privilege.
