## Design StackOVerflow

### 1. The Write Path (Postgres as Source of Truth)

Postgres is your Primary Record.

Transactions: When a user posts a question or logs in, it goes to Postgres first. This ensures that if the system crashes, your "Source of Truth" is ACID-compliant and safe.

Login & Auth: Keep this strictly in Postgres. OpenSearch isn't designed for the high-security, high-consistency needs of session management or password hashes.

The Sync Problem: How does the data get from Postgres to OpenSearch?

* The Cheap Way: Application-level dual-writes (write to DB, then write to Search). Warning: If the search write fails, your index is now out of sync.
* The Pro Way (CDC): Use Change Data Capture (CDC) with a tool like Debezium. It reads the Postgres WAL (Write Ahead Log) and streams updates to OpenSearch via Kafka. This is the gold standard for keeping them in sync without slowing down the user's request.

### 2. OpenSearch for the "Heavy Lifting"

OpenSearch (or Elasticsearch) handles everything that Postgres struggles with:
* **Fuzzy Search:** Handling "Pythno" when the user meant "Python."
* **Aggregations:** "Show me the top 5 tags in the last 24 hours." OpenSearch does this in milliseconds; Postgres would need a massive sequential scan.
* **Ranking:** Using "Relevance Scoring" (BM25) to ensure the best answer is at the top, not just the most recent one.

## Why use Postgres at all
Why cant all queries be answered using opensearch, only user credentials, counts etc can be in postgres

1. ACID vs. Eventual Consistency   
Postgres is built for ACID (Atomicity, Consistency, Isolation, Durability). OpenSearch is built for Speed.
    * The Scenario: You upvote a post, earn a "Great Answer" badge, and your reputation increases.
    * The Postgres Way: All three things happen inside a single transaction. If one fails, they all roll back. Your data is always correct.
    * The OpenSearch Way: OpenSearch is eventually consistent. If you wrote everything there, a user might upvote a post, but the "reputation" change might not show up for a second, or worse, if a node crashes during a refresh, that specific update could be lost or partially applied.

2. The Cost of "Joining" Data  
Stack Overflow is highly relational (User -> Post -> Comment -> Vote).
    * Postgres: Can perform complex JOIN operations efficiently using foreign keys. If a user changes their username, you update it in one place.
    * OpenSearch: It is essentially a flat-file system. To make it fast, you have to denormalize data (store the username inside every single post document).
    * The Headache: If a user changes their name, you now have to find and update every single document they ever touched in OpenSearch. That is an expensive, slow, and error-prone "Update by Query" operation.

3. Data Integrity & Constraints  
Postgres acts as the "Police Officer" of your data.
    * Schema Enforcement: Postgres ensures a post_id must exist before a comment can be attached to it (Referential Integrity).
    * OpenSearch: It’s "Schema-on-write." It’s very easy for "junk" data to creep in because the system won't stop you from writing a comment that points to a non-existent post. Over time, your search index becomes a "data swamp."

4. Rebuilding the Index  
Indexes get corrupted, or you might want to change how you tokenize text (e.g., changing how you handle C# vs C).
    * The Safety Net: If Postgres is your "Source of Truth," you can wipe your OpenSearch index and rebuild it from the SQL data in a few hours.
    * The Risk: If OpenSearch is your only database and the index gets corrupted or a mapping becomes incompatible, you have no backup to restore from. You lose the community's entire history.

| Feature| Postgres (The Source) | OpenSearch (The View) |
|---|---|---|
| User Profile & Rep	| Primary (Must be 100% accurate)|	No need to store in opensearch|
| Post Content |	Primary (The "Master" copy) |	Indexed (For searching words) |
| Votes & Counts| Primary (For audit/integrity)| no need to store in opensearch |
| The "Source of Truth" |	Yes |	No |

# Tradeoffs
By keeping UserProfiles exclusively in Postgres and Questions/Answers in OpenSearch, we avoid the "Update Storm" (where changing a username requires updating millions of documents in OpenSearch). However, as with every "perfect" design, there's a hidden tax you have to pay.

1. The Composition Flow  
When a user requests a page (e.g., "View Question"), your Orchestration Layer (API Gateway or a dedicated service) performs a parallel fetch:
    - Call A: Hits OpenSearch to get the Question text and Answer IDs.
    - Call B: Hits Postgres (or a Cache) to get the Profile Pictures and Usernames for the owner_id returned by Call A.
    - The Merge: The API layer stitches these together into one JSON response for the frontend.

2. The "N+1" Latency Problem
    - The Scenario: You are viewing a thread with 50 different answers from 50 different users.
    - The Trap: If your API composer fetches the OpenSearch data and then makes 50 separate calls to Postgres to get each user's profile, your page load time will skyrocket.
    - The Fix: You must use Batching. Your API composer collects all the unique user_ids from the OpenSearch result and makes one bulk query to Postgres: SELECT * FROM users WHERE id IN (1, 2, 3...50).

3. The "Sorting & Filtering" Tradeoff
    - The Problem: What if the user wants to "Search for questions about Java, but only from users with >10,000 reputation"?
    - The Conflict:
        - The "Java" part is in OpenSearch.
        - The "Reputation" part is in Postgres.
    - The Failure: OpenSearch can't filter by reputation because it doesn't have that data. Postgres can't filter by "Java" because it doesn't have the text index.
    - The Solution: You are forced to either:
        - Over-fetch: Get all Java questions from OpenSearch and filter them in the API layer (terrible for performance).
        - Denormalize slightly: Put only the "Reputation" field into OpenSearch while keeping the rest of the profile in Postgres.

**But if the reputation of a user changes we cant update million of documents**  
Instead of putting the exact reputation number in OpenSearch, you can use Reputation Buckets.
    - The Concept: Instead of reputation: 10452, you store rep_tier: 4.
    - The Logic: Most users want to filter by "High Reputation" or "Trusted Users," not specific numbers. Tiers (e.g., 1k+, 10k+, 50k+) change much less frequently than the raw score.
    - Result: You only update OpenSearch documents when a user "levels up" to a new tier, reducing your total updates by 99%.



## Understanding BM25 (Best Matching 25)
BM25 is an evolution of the classic TF-IDF (Term Frequency-Inverse Document Frequency) approach. It calculates a score for each document based on how well it matches your search query.

### The Three Pillars of BM25

BM25 looks at three specific factors to calculate a score:
1. Term Frequency (TF) — "The more, the merrier (up to a point)"

    Concept: If the word "Java" appears 10 times in a document, that document is likely more relevant than one where it appears once.

    The BM25 Twist (Saturation): Unlike older models, BM25 realizes that a word appearing 100 times isn't 100 times more important than a word appearing 10 times. It uses Term Frequency Saturation. After a few occurrences, the "extra" importance of seeing the word again starts to diminish.

2. Inverse Document Frequency (IDF) — "Rare is valuable"

    Concept: If you search for "The Python Tutorial," the word "The" appears in almost every document. It’s useless for ranking. The word "Python" is rarer and thus much more important.

    Calculation: BM25 heavily penalizes common words and rewards documents that contain the rare, "high-signal" keywords from your query.

3. Document Length Normalization — "Short and sweet"

    Concept: A 50-page book that mentions "React" 5 times is probably less focused on React than a 2-paragraph snippet that mentions it 5 times.

    Calculation: BM25 rewards shorter documents that contain your keywords because they have a higher "keyword density." It effectively "penalizes" long-winded documents where your search terms might just be a passing mention.

**BM25 is just TF-IDF with a ceiling on term frequency and a penalty for being too wordy**

Both OpenSearch and Solr (and Elasticsearch) are built on top of the same engine: Apache Lucene.

For a long time, TF-IDF was the default. However, around 2016, Lucene switched its default similarity algorithm to BM25. So today, whether you use OpenSearch or Solr, you are almost certainly using BM25 by default unless you've manually toggled it back to legacy TF-IDF.