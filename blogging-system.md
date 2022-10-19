# Design a blogging system 

Design a simple multi-user blogging platform, allowing writers to publish and manage the blogs under their personal publication and readers to read them.

### Functional and Non-functional Requirements

Functional Requirements
1. writers should be able to publish a public blog (editing out of scope for now)
2. readers should be able to read the blog
3. blog posts should be searchable
4. a user can be both - a reader as well as a writer
5. The blog may contain text, images, videos
6. The writer/reader can see the blogs published for a user
7. Draft etc can be out-of-score for now (but can be easily handled via a DB flag down the line, same for deletion/unpublish)
8. Comments are out of scope for now but will be good to discuss

Non-functional Requirements
1. Minimum latency
2. the platform should be scaled for 5 million daily active readers
3. the platform should be scaled for 10,000 daily active writers
4. Durable data, Strong consistency

### Database

Thinking about data requirements, we can go for either SQL database or even a No-SQL database.
Lets consider the pros and cons of both scenarios before honing in on one system

For our use case 
1. 10K daily active writers * 365 days * 10KB data per blog -> 34 GB of data per year
    a. This can be handled by SQL also, and we dont need joins here, so the data can be sharded and scaled up
2. For searching, we would need to support fuzzy search, hence **a separate ElasticSearch cluster** is essential
3. For lookups based on userid-post date combination, there is a simple Query operation on the DB
4. However, when we want to list all the blogs of a writer, we need to do Scan operations.
   1. for this, in nosql DBs like DynamoDB, we can design the "posts" table, such that
      a. UserId is the primary key, and DatePublished is a sort key
   2. In SQL databases, we can have the "posts" table containing userid(FK), post_date, and content such that
      a. we can simply query over the posts table with userid in where clause.
      b. remember to add an index over the userid column in post table to improve query performance
      c. obviously with the scale of data we are talking about the performance of the index might be a constraint on the performance

I would bat for dynamoDB here, because of  
1. the sheer flexibility of guaranteed constant performance even if the scale of data massively increases
2. Schemaless: with sql database the schema has to be predetermined before implementation

##### eg. the videos column:  
Obviously,  videos and images would be stored in a CDN (cloudfront services by a s3 bucket eg)    
However we might want to serve multi-resolution videos, based the reader device down the line.  
For this we can simply add columns to the post table and the right columns containing the video location can be picked up by our api


# APIs

* The API to fetch the data for a blog post is simple enough, as it involves a DB lookup
* However the API to put a blog post can write entries to dynamoDB as well as the Elastic Search cluster
* The API to search directly interacts with the Elastic Search cluster
