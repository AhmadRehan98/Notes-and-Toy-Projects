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

-
