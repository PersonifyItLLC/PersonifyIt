# Technical Decisions
I personally build PersonifyIt as a practical serverless product, so most of my technical choices were driven by access patterns, cost, iteration speed, and operational simplicity.

## Why did I pick this database?
For more context, PersonifyIt's core operations include the following:
- Fetching user profile by user ID or Cognito ID
- Checking / updating a user's credit balance
- Getting a design by user ID and design ID
- Listing designs for one user
- Listing designs inside one collection
- Listing public designs

Most databases are fine to support those following operations, so it wasn't about capability. But I picked DynamoDB essentially for the following reasons:
1. First, it gave me fast, predictable low-latency key-based reads and writes, and it fit naturally with my serverless stack of
Lambda and API Gateway.
2. Second, it allowed me to iterate faster because it's schemaless at the item level -- I could add or drop optional fields
as the product evolved without running migrations. The tradeoff there is that I lose database-level type and constraint enforcement, and that
validation must be handled inside of the application code.
3. I didn't have complex relational database needs, so the lack of joins and SQL aggregations (COUNT, SUM, GROUP BY arbitrary ORDER BY) wasn't a problem.
The only place I touch two tables is listing designs in a collection: I query the CollectionDesign join table by collectionId, then use the designId and
designCreatorId on each row to BatchGetItem the Design table. I denormalized designCreatorId onto the join row on purpose, because the design table's
key is userId + id, so I need both the creator ID and design ID to batch-get without adding a GSI.

I did consider a relational database like AWS Aurora, but its main advantage is relational querying and joins, and I wasn't relying on those.
I'd be taking on schema definition and migration overhead without really using the upside. If the product ever became analytics-heavy or needed
complex relational queries, that's when I'd add a relational store or a separate analytics store.

## Why did I pick these data models and fields?
I started from figuring out the access patterns that the application needed, and then I designed each table's keys and fields to serve those patterns.

That drove the decision for every field.

What is really important is to focus on the frontDesignData and backDesignData fields inside of the Design table. These fields are there because of
cohesion. Everything describing one side of the card lives together, so the record is organized and the front/back are symmetric. It also maps cleanly
to how the frontend consumes it: when you read a design, both the frontDesignData and backDesignData would be mapped directly.

userDetails is included inside of the Design table for denormalization purposes. It's a conscious tradeoff. I duplicated the username onto the design
to make reads cheaper, and the cost of that is just a copy. If a user changed their username, the userDetails would be there and older designs could go stale.
username changes are rare though and the impact is minimal for now though.

## What happens when you need a new access pattern?
We may need a new GSI to an existing table and we may need to backfill existing items. We may need to create a new table.

## How would you account for scale?
We should go layer by layer when it comes to the resources we have. 

- With DynamoDB, it scales horizontally, and I would need to watch out for specific risks.
The biggest risk is a hot partition: my public feed queries the byPublic GSI where every public design shares the partition key "public." At 1000x, that's a hot key.
I'd shard it with something like "public#<bucket>" across N buckets, or time-bucketing by month. I'd keep capacity on on-demand, add a cache like DAX or a read-through
cache for very hot reads, and only add GSIs for proven patterns since each one adds write cost.
- With Lambda, it would scale out automatically. I would set reserved or provisioned concurrency on critical paths so a spike on one route can't starve others, add 
throttling / rate limits at API Gateway, and watch cold starts on hot routes. I'd also make sure a burst of traffic can't overwhelm the downstream systems.
- With S3 and CloudFront, the CDN will absorb the bulk of read traffic and it never touches compute or database. At 1000x, we should focus on cache hit rates. Tuning
TTLs and cache headers so origin (S3) fetches stay low. The queue decouples the client from the work. The job record decouples the work from the notification.
- The riskest path is the AI image generation. It currently costs money per call. At 1000x, we need to incorporate scale. There are two ways
that we can go about that:
1. We could have a job model where the client POSTs a job, gets a jobId, then polls GET /jobs/{id} until it's ready. This doesn't require a queue, but a job record in DynamoDB.
We could have the client open a persistent connection and when the worker sets the job to ready, the server pushes the update down that connection. 
2. We could have a queue and workers behind it. The queue is purely the scaling / processing upgrade; polling stays the notification mechanism. 
- No locks are going to be needed as credit spending uses an atomic conditional write.
- Stripe webhooks are idempotent via the byStripeSessionId index
- I would invest in observability -- dashboards, alarms, and tracing are important to find bottlenecks fast. 

