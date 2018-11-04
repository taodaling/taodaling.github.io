---
categories: tool
layout: post
---

{:toc}

# cluster

## health

TO check the cluster health, we will using the _cat API.

```http
GET /_cat/health?v
```

Whenever we ask for the cluster health, we either get green, yellow, or red.

- Green - everything is good.(cluster is fully functional)
- Yellow - all data is available but some replicas are not yet allocated(cluster is fully functional)
- Red - some data is not available for whatever reason(cluster is partially functional)

## nodes

We can also get a list of nodes in our cluster as follows:

```http
GET /_cat/nodes?v
```



## index

Let's take a look on our indices:

```http
GET /_cat/indices?v
```

# index

## create

Create an index named "customer" and then list all the indexes again:

```http
PUT /customer?pretty
```

## delete

Let's delete the index that we just created.

```http
DELETE /customer?pretty
```



# Document

## add

Let's now put something into our customer index. We'll index a simple customer document into the customer index, with an ID of 1 as follows:

```http
PUT /customer/_doc/1?pretty
{
  "name": "John Doe"
}
```

Remember in heart, elasticsearch doesn't require you to explicitly create an index first before you can index document into it, elasticsearch will automatically create the customer index if it didn't already exist beforehand.

## get

Retrieve that document that we just indexed:

```http
GET /customer/_doc/1?pretty
```

## update

```http
POST /customer/_doc/1/_update?pretty
{
  "doc": { "name": "Jane Doe" }
}
```

## delete

```http
DELETE /customer/_doc/2?pretty
```

## bulk operation

```http
POST /customer/_doc/_bulk?pretty
{"index":{"_id":"1"}}
{"name":"John Doe"}
{"update":{"_id":"1"}}
{"doc": {"name":"John Doe becomes Jane Doe"}}
{"delete":{"_id":"1"}}
```

# Search

retrieve partial document

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
```

In the previous request, we retrieve total document, but int the next request, we only need two fields of each document.

```http
GET /bank/_search
{
    "query":{"match_all":{}},
    "_source":["account_number", "balance"]
}
```

## match

Return the account numbered 20.

```http
GET /bank/_search
{
  "query": { "match": { "account_number": 20 } }
}
```

This example returns all accounts containing the term "mill" in the address:

```http
GET /bank/_search
{
  "query": { "match": { "address": "mill" } }
}
```

This example returns all accounts containing the term "mill" or "lane" in the address:

```http
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

## match_phrase

This example returns all accounts containing the phrase "mill lane" in the address:

```http
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

## bool

This example compose two match queries and returns all accounts containing "mill" and "lane" in the address:

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

This example compose two match queries and returns all accounts containing "mill" or "lane" in the address.

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

This example composes two match queries and returns all accounts that contain neither "mill" nor "lane" in the address:

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

We can combine must, should, and must_not clauses simultaneously inside a bool query. Furthermore, we can compose bool queries inside any of these bool clauses to mimic any complex multi-level boolean logic. These bool queries are concatenate with `and`.

## filter

For _score field in the search results, the score is a numeric value that is a relative measure of how well the document matches the search query that we specified. The higher the score is , the more relevant the document is.

But queries don't always need to produce scores, in particular when they are only used for "filtering" the document set. Elasticsearch detects these situations and automatically optimizes query execution in order not to compute useless scores.

The bool query that we introduced in the previous section also supports filter clauses which allow to use a query to restrict the documents that will be matched  by other clauses, without changing how scores are computed. 

### range

This example uses a bool query to return all accounts with balances between 20000 and 30000, inclusive.

```js
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

 ## Aggregations

Aggregations provide the ability to group and extract statistics from your data. The easiest way to think about aggregations is by roughly equating it to the SQL GROUP BY and the SQL aggregate functions. In Elasticsearch, you have the ability to execute searches returning hits and at the same time return aggregated results separate from the hits all in one response. This is very powerful and efficient in the sense that you can run queries and multiple aggregations and get the results back of both operations in one shot avoiding network roundtrips using a concise and simplified API.

### group

This example groups all the accounts by state, and then returns the top 10 (default) states sorted by count descending (default).

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
```

In SQL, the above aggregation is similar in concept to:

```sql
SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC LIMIT 10
```

### average

This example calculates the average account balance by state.

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

## range

This example demonstrates how we can group by age brackets(ages 20-29, 30-39, 40-49), then by gender, then get the average account balance.

```http
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
```