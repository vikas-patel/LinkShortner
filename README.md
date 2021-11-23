# Url Shortener
The backend project is to design, plan, and estimate building a simple link shortening service (think very simple bit.ly).

## Functional Requirements
1. Generate short URL from a long URL
2. Redirect user to long URL when access to Short URL
3. Analytics of link shortening service
4. REST API to expose URL functionality

## Non Functional Requirement
1. Scalable
2. Low Latency
3. Minimum length of short URL

## Estimates:
#### Traffic:
* daily link redirection: 100k
* daily links generated: 10k

Number of unique shortened links generated per seconds = 10k /(24 hours * 3600 seconds ) ~ 7 URLs/minutes

Number of redirections = 100k /(24 hours * 3600 seconds ) = 70 URLs/minutes

### DB Size:
1k daily users -> 1k * 5-10 users * 30 days * 12 months * 10 years

36M records in database

1 record : 2Kb (max long url size 2000 characters)

DB Size: 2kb * 36M = 72 GB

### Memory:
Assume 20% of daily data (frequently accessed) is cached.

Cache memory: 0.20 (cache percentage) * 10k (daily generated urls) * 2kb (max size of url) * 30 days * 12 months

= 1.4 GB

## High Level Design
To ensure system is scalable both data and traffic; and high availability:
1. Added a load balancer in front of WebServers
2. Sharded the database to handle huge object data
3. Added cache system to reduce load on the database.

![Scalable High Level Design](https://raw.githubusercontent.com/vikas-patel/LinkShortner/main/Link%20Shortner.png)

## Database?
RDBMS could be good choice but it will have problems when database size and traffic grows, altough we can shard but this would increase the complexity of design.
I would personally prefer NoSQL database. Although they are eventually consistent but easy to scale.

## Schema:
### URL
* hash_url (primary key)
* original_url
* created_date
* expiration_date
* customer_id

### CUSTOMER
* user_id (Primary key)
* name
* email
* created_date

## REST API
* POST create( long_url, customer_id)

Return Value: The short Url generated, or error code in case of the inappropriate parameter.

* GET: /{short_url}

Return Value: The original long URL, or invalid URL error code.

Set http status code 302

returning 302 redirect will ensure all requests for redirection reaches to our backend and we can perform analytics (Which is a functional requirement)

## Shortening Algorithm
## URL Length
What is the minimum length of the shortened URL to represent 36M URLs? We will use 62 chars (a-z, A-Z, 0–9) to encode our URLs which leads us to use base62 encoding.

62 power 4 > 36,000,000

4 chars length short url should be enough to represent 36M urls.

## Encoding
1. Join the original key and customer_id and then hash it, so generated short url will unique for each customer.
We can compute it using the Unique Hash(MD5, SHA256, etc.) and then encode using base62. If we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base62 encoding, we’ll get a string having more than four characters. We can take the first 4 characters for the key.

2. The first 4 characters could be the same for different long URLs so check the DB to verify that TinyURL is not used already

3. Try next 4 characters of previous choice of 4 characters already exist in DB and continue until you find a unique value

## DB Scale
* MongoDB supports distributing data across multiple machines using shards. Since we have to support large data sets and high throughput, we can leverage sharding feature of MongoDB.
* Since MongoDB supports transaction for a single document, we can maintain consistency and because we are using hash based shard key, we can efficiently use putIfAbsent (As a hash will always be corresponding to a shard)
* To speed up reads (checking whether a Short URL exists in DB or what is Original url corresponding to a short URL) we can create indexing on ShortURL.

### DB Read Speedup
#### Cache

We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like Memcached, which can store full URLs with their respective hashes. Before hitting backend storage, the application servers can quickly check if the cache has the desired URL.

#### Scale Horizontally - Load Balancer

We could use a simple Round Robin approach that distributes incoming requests equally among backend servers. This LB is simple to implement and does not introduce any overhead. Another benefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop sending any traffic to it.

## Analytics
1. How many times a short URL has been used? 
2. most visited links ?
3. aggregate over day, week, month?

I don't like to write in-house code for generic functionality, better use external third party analytics providers e.g. Segment or Firebase Big Query

Each time tiny url request reaches service’s backend. We can push this data(tiny url, users etc.) to a temporary storage and periodically send this data to external third-party analytics providers.

## Estimates
### Development - 18 days
*	1d - Initial project setup (Web Framework and Database)
*	4h - DO (Database Object)
*	4h - DAO Layer (Databaase Access Layer)
*	3d - Shortening Services
*	2d - Cache Service
*	2d - Routing API
*	2d - Analytics API
*	3d - Error Handling
*	4d - Scale up (load balancer, DB Shard)
### Write unit tests
*	6 days - 30% of development
### Functional Testing and Bug fixes
*	9 days - 50% of development
### Load Testing
*	3 days
### Deployment
*	3 days
### Buffer Time
*	8 days - 20% of overall project
### Go Live Date
* Total working days - 47 days
* Holidays - 2 days
* Meetings - 1 day
* Estimated day go live date - 50 days /(5 days a week) weeks from now.

10 weeks for a single contributor
