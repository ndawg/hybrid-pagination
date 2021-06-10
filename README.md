Motivation
---
There are two main pagination strategies: offset based and cursor based. Each of these methods has certain pros and cons (which have been discussed ad nauseam), but briefly:

**Offset based**
- ✔️ Data is in order when single threaded
- ✔️ Can be multithreaded
- ❌ Inefficient DB queries when using `OFFSET`
- ❌ Will miss results (or send duplicates) if data shifts

**Cursor based**
- ✔️ Will never miss results when data shifts
- ✔️ Data is always in order
- ✔️ Efficient DB queries
- ❌ Cannot be multithreaded, requires waiting for response

There are two principal trade-offs: speed versus integrity. In some applications, it's acceptable to miss some data; in others, it's acceptable for the requests to take a long time. What about a situation where you need *both* speed and consistency? Consider an ETL: "extracting" and "loading" the data to and from a database (often via an HTTP API endpoint) can take an enormous amount of time, but you really don't want to miss any data.

Hybrid Approach
---

In order to fix this, I propose a hybrid multi-cursor approach that works as follows:

1. The client requests *n* cursors from the server. This number is equivalent to the number of threads that the client will use in parallelization (usually <= 10).
2. The server divides the records using a unique, sequential, permanent ID field into *n* buckets. From this, it creates *n* cursors that include the lower bound, upper bound, and the next ID to return (which defaults to the lower bound).
3. The client can use each cursor to make requests (which will always yield a new cursor, just like normal cursor pagination). The lower and upper bound of each cursor will never change. In this way, each cursor is confined to its own bucket of the data.
4. The server can retrieve records from the database using a simple `BETWEEN next AND upper` query (using the data from the cursor) instead of relying on expensive offsets queries.

In addition, the *n*th cursor can represent a special bucket that has no upper bound. This allows the client to receive any trailing records that might not have existed when the data began to be pulled. However, this is optional (see note on "perfect pulling").

Example
---
Consider a database with 10,000 records. Pulling these records with a page size of 50 results in 200 queries. If each query takes 500ms, it will take a minute and a half to pull all records when using cursor pagination. Not great! What if we use the hybrid approach?

First, let's assume the client will use 5 threads. The server will divide our data into 5 buckets:

| bucket | ID range         | cursor |
| ------ | -----            | ------- |
| 1      | `[0, 2000)`      | `{lower: 0,    upper: 2000,  next: 0}`
| 2      | `[2000, 4000)`   | `{lower: 2000, upper: 4000,  next: 2000}`
| 3      | `[4000, 6000)`   | `{lower: 4000, upper: 6000,  next: 4000}`
| 4      | `[6000, 8000)`   | `{lower: 6000, upper: 8000,  next: 6000}`
| 5      | `[8000, 10000)`  | `{lower: 8000, upper: 10000, next: 8000}`

Next, let's take a look at what a single bucket's request cycle looks like. Bucket 1 will make a request with its cursor:
```
GET sample.api/records?lower=0&upper=2000&next=0&size=50
```

The API can easily get the results from the database. Note that if a record inbetween `next` and `upper` has been deleted, this will not cause any duplicate return (since IDs are permanent):
```sql
SELECT * FROM table WHERE table.id BETWEEN next AND upper LIMIT size;
```

The API will return both the results as well as a new cursor, which bucket 1 will then send back in the next request:
```json
{
    "results": { ... },
    "cursor": {
        "lower": 0,
        "upper": 2000,
        "next": 50
    }
}
```
Notice that the lower and upper bounds of the cursor don't change, but the anticipated value does. That way, when the next query is made, the window will be shifted - however, the cursor will never reach beyond its upper limit, so no duplicates will ever be returned. This allows us to parallelize the operation. 

Revisiting our earlier calculations, each bucket is responsible for pulling 2000 records: 2000 records / 50 per page * 500ms per request = 20 seconds. Since everything is running in parallel, the entire operation should take 20 seconds! Much better:

- ✔️ Will never miss results when data shifts
- ✔️ Can be multithreaded
- ✔️ Efficient DB queries
- ❌ Data is always in order

Data
---
In real world tests, this showed significant improvements in performance.

| records | cursor | hybrid (5 threads) | diff |
| ------------ | ------ | ------ | ---- |
| 1,936        | 35,347ms | 8,526ms | -27 secs (74% improvement)
| 9,643        | 191,003ms | 38,931ms | -2.5 mins (80% improvement)
| 96,269       | 1,829,111ms | 385,076ms | -24 mins (82% improvement)

Notes
----
**"Perfect" pulling**

Oftentimes, the offset based pagination is dismissed because it might miss certain records (or receive a duplicate). Depending on the usage, this is probably more okay than one might assume. Consider that even with cursor based pagination (and the hybrid approach), a record that has already been retrieved might be deleted by the time the requests are done - ie, no pagination method will ensure the client receives a 100% up-to-date snapshot of the data.

**Bias**

One tricky consideration is distribution of IDs. If data is deleted often, there might be significant and unequal gaps in different buckets. This could reduce the effectiveness.

As an added benefit, however, the divison of buckets can be modified to favor certain ranges of data. For example, in a chat application, you might want to retrieve newer messages much quicker. Buckets associated with the newest data can be smaller to make pulling quicker when multithreading.

**Lower bound**

There's no need to include the lower bound in the cursor (`next` can be used instead), but it makes it easier to explain the concept of buckets.
