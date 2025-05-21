---
layout: post
title: Running AWS DocumentDB on a Budget -  Real-World Experience
---


When we first decided to migrate our MongoDB workloads to the cloud, cost was a significant concern. AWS DocumentDB looked promising with its MongoDB-compatible API, but its reputation for being expensive gave us pause. After running production workloads on it for the past year, I wanted to share our experience running DocumentDB on a budget and some important lessons we learned along the way.

## Medium Tier: Surprisingly Capable

One of our first pleasant surprises was discovering that DocumentDB's medium tier instances are actually quite capable. Amazon's marketing pushes customers toward larger instance types, but we found that `db.r5.large` or even `db.r5.medium` instances handled our production workloads remarkably well.

Our application serves approximately 50,000 daily active users, with peak loads generating around 1,200 operations per second. The medium tier instances maintained sub-10ms response times for most operations, only occasionally spiking during heavy write operations. 

We started with a three-node cluster (one primary, two replicas) using `db.r5.medium` instances, which provided enough redundancy while keeping our monthly compute costs under $500. This was significantly less than the $1,500+ we had initially budgeted based on AWS's own sizing recommendations.

## Separating Storage and Compute: The Hidden Cost Advantage

Perhaps the biggest cost advantage of DocumentDB comes from its architecture, which separates storage from compute. This architectural choice pays dividends in several ways:

1. **Pay only for storage you use**: Unlike self-managed MongoDB where you provision entire volumes, DocumentDB charges only for actual data stored plus some metadata.

2. **No storage provisioning headaches**: We never have to worry about running out of disk space or over-provisioning expensive SSD volumes.

3. **Automatic storage scaling**: As our database grew from 50GB to over 500GB, we didn't need to modify our infrastructure or suffer downtime for storage expansion.

This separation allowed us to optimize compute and storage independently. During our busiest season, we temporarily scaled up to `db.r5.large` instances for additional CPU and memory, then scaled back down once the traffic normalized—all without any storage migration or downtime.

This flexibility alone saved us approximately 40% compared to our previous MongoDB Atlas configuration, where we had to provision instances large enough to accommodate both our compute requirements and storage projections.

## MongoDB Compatibility: The Good, the Bad, and the Ugly

AWS advertises DocumentDB as "MongoDB-compatible," which is true to an extent. However, we encountered several compatibility issues that required application changes:

### Missing Operators and Data Type Conversion

The most significant pain point was discovering that DocumentDB doesn't support certain MongoDB operators we relied on. For example, the `$lookup` operator (MongoDB's equivalent of a JOIN) has limitations in DocumentDB, and some array operators like `$all` behave differently.

Additionally, DocumentDB is much stricter about data types than MongoDB. Where MongoDB would automatically convert strings to numbers or ObjectIds when appropriate, DocumentDB requires exact type matching. This caused several subtle bugs during our migration that were difficult to track down.

We suspect these differences stem from DocumentDB's underlying implementation, which uses a modified Aurora PostgreSQL database rather than MongoDB's native storage engine. While this gives AWS better control over the service, it creates these compatibility gaps.

We ended up creating a compatibility layer in our application that detects when it's connecting to DocumentDB and modifies queries accordingly. It wasn't ideal, but it allowed us to maintain a single codebase that works with both databases.

### Collection Name Length Restrictions

One unexpected issue we faced was DocumentDB's stricter limitations on collection name lengths. Our application uses dynamically generated collection names that include GUIDs to partition data for different customers. These names frequently exceeded DocumentDB's limits.

We had to re-engineer our collection naming strategy to use shorter, hashed identifiers instead of full GUIDs. This required a complex migration process and changes to our application logic, taking our team about three weeks to fully implement and test.

If you're planning a migration, carefully review DocumentDB's [limits and restrictions](https://docs.aws.amazon.com/documentdb/latest/developerguide/limits.html) documentation—it might save you some painful surprises.

## Performance Optimization: Working Around Cursor Limits

On the medium tier, we occasionally hit cursor limitations when running complex analytical queries that return large result sets. DocumentDB would terminate the cursor before returning all results, causing our reporting jobs to fail.

However, we found that these issues could almost always be resolved with better query optimization and proper indexing. For instance:

- Adding compound indexes to support our most common query patterns
- Rewriting queries to include more specific filters
- Using the `$project` operator to return only necessary fields
- Implementing pagination in our application logic

After applying these optimizations, our cursor issues virtually disappeared, and we saw a 30-40% improvement in query performance as a bonus. This experience reinforced that right-sizing your database often involves optimizing your application as much as choosing the correct infrastructure.

## Slow Version Updates: The Patience Game

If there's one area where the budget-friendly approach to DocumentDB really shows its limitations, it's with version updates. AWS is notoriously slow to implement new MongoDB features in DocumentDB. As of this writing, DocumentDB still doesn't fully support all MongoDB 4.0 features, while MongoDB itself is already on version 7.0.

Additionally, applying version updates to your DocumentDB cluster is a time-consuming process that requires patience. Our recent update from 4.0 to 5.0 compatibility took nearly 6 hours to complete, even though our data volume is relatively modest.

If your application depends on bleeding-edge MongoDB features, DocumentDB might not be the right choice. But if you can work within its compatibility level, the cost savings can be substantial.

## The Bottom Line: Cost vs. Compromise

After a year of running production workloads on DocumentDB, our verdict is that it's a viable budget-friendly option for MongoDB workloads with some caveats.

Here's our rough monthly cost breakdown for a medium-sized application:
- 3 × db.r5.medium instances: ~$450
- Storage (500GB): ~$60
- I/O operations: ~$90
- Backups: ~$30
- **Total monthly cost**: ~$630

For comparison, a similarly configured MongoDB Atlas cluster would cost us approximately $1,100 per month.

Is the 40% cost reduction worth the compatibility compromises and engineering work? For our team, it was. Your mileage may vary depending on how deeply your application leverages MongoDB-specific features.

## Recommendations for Budget-Conscious Teams

If you're considering DocumentDB as a cost-saving measure, here are my recommendations:

1. **Start with medium tier instances** instead of following AWS's aggressive sizing recommendations
2. **Thoroughly test your application** against DocumentDB before migrating to identify compatibility issues
3. **Invest time in proper indexing** to avoid performance problems on smaller instances
4. **Consider a compatibility layer** in your application if you need to support both MongoDB and DocumentDB
5. **Take advantage of the storage/compute separation** by sizing instances for your compute needs, not storage projections

With the right approach and expectations, DocumentDB can deliver solid performance at a reasonable cost—just be prepared to roll up your sleeves and make some adjustments along the way.

