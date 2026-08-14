# Data Processing (8.15)

**8.15 objectives covered here:**
1. Define a mapping that satisfies a given set of requirements
2. Define and use multi-fields with different data types and/or analyzers
3. Use the Reindex API and Update By Query API to reindex and/or update documents
4. Define and use an ingest pipeline that satisfies a given set of requirements
5. Define runtime fields to retrieve custom values using Painless scripting :sparkles: *(new since 8.1)*

> :books: "Define and use a custom analyzer" and "Configure an index so that it properly maintains the relationships of nested arrays of objects" are **no longer separate objectives** in 8.15. Custom analyzers are still needed for the multi-fields objective so they stay inline below; nested objects are moved to a clearly-marked bonus section at the bottom.

---

#  Define a mapping that satisfies a given set of requirements

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping.html
> Mapping is the process of defining how a document, and the fields it contains, are stored and indexed.

> Each document is a collection of fields, which each have their own data type. When mapping your data, you create a mapping definition, which contains a list of fields that are pertinent to the document. 

> A mapping definition also includes metadata fields, like the `_source` field, which customize how a document’s associated metadata is handled.


## Dynamic mapping
> Dynamic mapping allows you to experiment with and explore data when you’re just getting started. Elasticsearch adds new fields automatically, just by indexing a document. You can add fields to the top-level mapping, and to inner object and nested fields.

## Explicit mapping
> Explicit mapping allows you to precisely choose how to define the mapping definition, such as:
> 
> - Which string fields should be treated as full text fields.
> - Which fields contain numbers, dates, or geolocations.
> - The format of date values.
> - Custom rules to control the mapping for dynamically added fields.

:question: 1. Create a new index mapping called `henry4` that matches the following requirements:

- speaker: keyword
- line_id: keyword and not aggregateable 
- speech_number: integer

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-types.html


### Mapping numeric identifiers
>Not all numeric data should be mapped as a numeric field data type. Elasticsearch optimizes numeric fields, such as integer or long, for range queries. However, keyword fields are better for term and other term-level queries.
>
>Identifiers, such as an ISBN or a product ID, are rarely used in range queries. However, they are often retrieved using term-level queries.
>
>Consider mapping a numeric identifier as a keyword if:
>
> - You don’t plan to search for the identifier data using range queries.
> - Fast retrieval is important. term query searches on keyword fields are often faster than term searches on numeric fields.
> 
> If you’re unsure which to use, you can use a multi-field to map the data as both a keyword and a numeric data type.
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/doc-values.html
> All fields which support doc values have them enabled by default. If you are sure that you __don’t need to sort or aggregate__ on a field, or access the field value from a script, you can disable doc values in order to save disk space


```json
PUT /henry4
{
 "settings": {
   "number_of_replicas": 0
 },
 "mappings": {
   "properties": {
    "speaker": {"type": "keyword"},
    "line_id": {
      "type": "keyword",
      "doc_values": false
    },
    "speech_number": {"type": "integer"}
  }
 }
}
```

</details>
<hr/>

:question: 2. Using the previous ingested `shakespeare` index, re-index the data into the new one called `henry4` that only contains the lines for the play `Henry IV`

<details>
  <summary>View Solution (click to reveal)</summary>

The best way to do this is to write the term query first, check that contains what you want, then convert the query into the `_reindex`

```json
POST _reindex
{
  "source": { "index": "shakespeare",
    "query": {
      "term": {
        "play_name": "Henry IV"
      }
    }
  },
  "dest":   { "index": "henry4" }
}

// output

{
  "took" : 2844,
  "timed_out" : false,
  "total" : 3205,
  "updated" : 0,
  "created" : 3205,
  "deleted" : 0,
  "batches" : 4,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```
</details>
<hr/>

:question: 3. verify that the data only contains `HENRY IV` play lines.

<details>
  <summary>View Solution (click to reveal)</summary>

If only one play is present, a terms aggregation on `play_name` returns exactly one bucket:

```json
GET henry4/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "plays": { "terms": { "field": "play_name", "size": 20 } }
  }
}

// output - one bucket, doc_count matching the reindex "created" count
{
  "aggregations": {
    "plays": {
      "buckets": [ { "key": "Henry IV", "doc_count": 3205 } ]
    }
  }
}
```

Or prove the negative:
```json
GET henry4/_count
{
  "query": {
    "bool": { "must_not": [ { "term": { "play_name": "Henry IV" } } ] }
  }
}
// expect "count": 0
```
</details>
<hr/>

## Mapping parameters worth memorising

The wording of "satisfies a given set of requirements" maps almost one-to-one onto these:

| Requirement wording | Mapping parameter |
| --- | --- |
| "not aggregatable / not sortable" | `"doc_values": false` |
| "searchable but not returned" / "do not index" | `"index": false` |
| "must not be stored in `_source`" | `"_source": { "excludes": ["field"] }` |
| "ignore values longer than N" | `"ignore_above": N` |
| "do not reject malformed values" | `"ignore_malformed": true` |
| "accept these date formats" | `"format": "yyyy-MM-dd||epoch_millis"` |
| "use this value when the field is missing" | `"null_value": "NONE"` |
| "field should be searchable by two different names" | `"copy_to": "combined_field"` |
| "reject unknown fields" | `"dynamic": "strict"` at the mapping root |
| "count of occurrences should not affect score" | `"norms": false` / `"index_options": "docs"` |
| "must be an exact match, case-insensitive" | `keyword` + a `normalizer` |

## Changing a mapping after the fact

```json
// Adding a NEW field is allowed:
PUT henry4/_mapping
{ "properties": { "act": { "type": "keyword" } } }

// Changing an EXISTING field's type is NOT.
// The answer is always: create a new index with the right mapping, then _reindex.
```

Useful checks:
```json
GET henry4/_mapping
GET henry4/_mapping/field/line_id
GET henry4/_field_caps?fields=*
```

---

#  (Background) Define and use a custom analyzer

and

# Define and use multi-fields with different data types and/or analyzers
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-types.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analyzer.html  <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-standard-analyzer.html  <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-custom-analyzer.html#_configuration

:question: 1. Write a custom analyzer that changes the name of `PRINCE HENRY` to `WAYWARD PRINCE HAL` in the `speaker` field, add this to a new index called `henry4_hal`

- Create a new index, 
- with a mapping on the speaker field 
- that utilises an analyser 
- to rename the princes name if it matches.

<details>
  <summary>View Solution (click to reveal)</summary>


## Test the analyser

```json
POST _analyze
{
  "char_filter": {
      "type": "pattern_replace",
      "pattern": "PRINCE HENRY",
      "replacement": "WAYWARD PRINCE HAL"
  },
  "text": [
    "PRINCE WILLIAM",
    "PRINCE HENRY",
    "PRINCE HARRY"
  ]
}

// output

{
  "tokens" : [
    {
      "token" : "PRINCE WILLIAM",
      "start_offset" : 0,
      "end_offset" : 14,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "WAYWARD PRINCE HAL",
      "start_offset" : 15,
      "end_offset" : 27,
      "type" : "word",
      "position" : 101
    },
    {
      "token" : "PRINCE HARRY",
      "start_offset" : 28,
      "end_offset" : 40,
      "type" : "word",
      "position" : 202
    }
  ]
}

```

## Put it all together

- mappings -> `speaker` -> `"analyzer": "wayward_son_analyser"`

- settings -> analysis -> `"wayward_son_analyser"` -> `"char_filter": ["rename_filter"]`

- `"rename_filter"` -> `"pattern_replace"`

```json
PUT /henry4_hal
{
  "mappings": {
    "properties": {
      "speaker": {
        "type": "text",
        "analyzer": "wayward_son_analyser",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "line_id": {
        "type": "integer"
      },
      "speech_number": {
        "type": "integer"
      }
    }
  },
  "settings": {
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "wayward_son_analyser": {
          "type":      "custom", 
          "tokenizer": "standard",
          "char_filter": [
            "rename_filter"
          ]
        }
      },
      "char_filter": {
        "rename_filter": {
          "type": "pattern_replace",
          "pattern": "PRINCE HENRY",
          "replacement": "WAYWARD PRINCE HAL"
        }
      }
    }
  }
}
```
</details>
<hr/>

:question: 2. re-index `henry4` into `henry4_hal`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST _reindex
{
  "source": { "index": "shakespeare",
    "query": {
      "term": {
        "play_name": "Henry IV"
      }
    }
  },
  "dest":   { "index": "henry4_hal" }
}
```
</details>
<hr/>

:question: 3. verify by querying the `henry4_hal` index for the speaker `HAL`, `WAYWARD` and `PRINCE HENRY`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET henry4_hal/_search
{
  "query": {
    "term": {
      "speaker": "HAL"
    }
  }
}
```

> NOTE: Oddly you can't see the `HAL` or `WAYWARD` in the returned data, but you can search for it.
> What you get returned is the original data `PRINCE HENRY`

:bulb: **This is expected, not a bug.** `_source` is the raw JSON you sent — analysers only affect the *inverted index*, never `_source`. So you search for `HAL` and match, but you get `PRINCE HENRY` back. Prove what was actually indexed with the termvectors API:

```json
GET henry4_hal/_termvectors/<doc_id>?fields=speaker
```
</details>
<hr/>

## Multi-fields — the actual 8.15 objective

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/multi-fields.html

> It is often useful to index the same field in different ways for different purposes. This is the purpose of multi-fields. Most field types support multi-fields via the `fields` parameter.

Three things a question can ask for, and they are frequently combined:

1. **Different data types** — `text` for search + `keyword` for sort/aggregate
2. **Different analyzers** — e.g. `standard` + `english` for stemming
3. **A normalizer** — a `keyword` sub-field that is lowercased for case-insensitive exact matching

:warning: Multi-fields **cannot be added retrospectively to existing documents' index data**. You can add the sub-field to the mapping, but existing docs are only searchable through it after a `_reindex` or an `_update_by_query` (which re-indexes each doc in place).

<hr>

:question: Create an index `plays` where the `title` field is:
- full-text searchable with the standard analyzer
- also searchable with English stemming, under `title.english`
- also sortable/aggregatable as an exact value, under `title.raw`
- also matchable case-insensitively as an exact value, under `title.ci`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT plays
{
  "settings": {
    "number_of_replicas": 0,
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "english": {
            "type": "text",
            "analyzer": "english"
          },
          "raw": {
            "type": "keyword",
            "ignore_above": 256
          },
          "ci": {
            "type": "keyword",
            "normalizer": "lowercase_normalizer"
          }
        }
      }
    }
  }
}
```

Test each path:
```json
POST plays/_doc?refresh
{ "title": "The Tragedy of Hamlet, Prince of Denmark" }

// stemming - "tragedies" matches "Tragedy" only via the english sub-field
GET plays/_search
{ "query": { "match": { "title.english": "tragedies" } } }

// exact value - full string, case sensitive
GET plays/_search
{ "query": { "term": { "title.raw": "The Tragedy of Hamlet, Prince of Denmark" } } }

// exact value, case insensitive
GET plays/_search
{ "query": { "term": { "title.ci": "the tragedy of hamlet, prince of denmark" } } }

// aggregate on the keyword sub-field
GET plays/_search?filter_path=aggregations
{ "size": 0, "aggs": { "titles": { "terms": { "field": "title.raw" } } } }
```

:bulb: Always verify an analyzer with `_analyze` before you commit to it:
```json
GET plays/_analyze
{ "field": "title.english", "text": "The Tragedy of Hamlet" }
```
</details>
<hr>

:question: You need a different analyzer at index time and at search time (index with an edge-ngram for autocomplete, search with the standard analyzer). How?

<details>
  <summary>View Solution (click to reveal)</summary>

`search_analyzer` overrides `analyzer` at query time. This is the classic autocomplete recipe — without it, the search terms get ngrammed too and everything matches everything:

```json
PUT autocomplete-idx
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngrams"]
        }
      },
      "filter": {
        "edge_ngrams": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 15
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete_index",
        "search_analyzer": "standard"
      }
    }
  }
}
```
</details>
<hr>

## Analyzer anatomy (needed for the multi-fields objective)

An analyzer is exactly three stages, in this order:

| Stage | Count | Examples |
| --- | --- | --- |
| `char_filter` | 0..n | `html_strip`, `mapping`, `pattern_replace` |
| `tokenizer` | exactly 1 | `standard`, `keyword`, `whitespace`, `pattern`, `edge_ngram`, `path_hierarchy` |
| `filter` (token filters) | 0..n | `lowercase`, `stop`, `stemmer`, `synonym`, `asciifolding`, `edge_ngram` |

Test any combination without creating an index:
```json
POST _analyze
{
  "char_filter": ["html_strip"],
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "<p>The QUICK brown Foxes</p>"
}
```

Test a named analyzer inside an existing index:
```json
POST my-index/_analyze
{ "analyzer": "my_custom_analyzer", "text": "some text" }
```



# Use the Reindex API and Update By Query API to reindex and/or update documents

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-reindex.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-update-by-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-delete-by-query.html

## Which API for which question

| Question says | Use |
| --- | --- |
| "copy / migrate data into a new index (with a new mapping)" | `_reindex` |
| "change the values of documents **in place**" | `_update_by_query` |
| "apply a new mapping to existing documents" (mapping already changed) | `_update_by_query` with no body — it just re-indexes each doc |
| "run documents through a pipeline" | either, with `?pipeline=` (UBQ) or `"dest": { "pipeline": "..." }` (reindex) |
| "remove documents matching X" | `_delete_by_query` |

## Options that come up

```json
POST _reindex?wait_for_completion=false&refresh=true
{
  "conflicts": "proceed",                 // don't abort on version conflicts
  "max_docs": 1000,                       // limit how many documents are processed
  "source": {
    "index": "accounts-raw",
    "query":  { "term": { "gender.keyword": "F" } },
    "_source": ["firstname", "lastname", "balance"],   // only copy some fields
    "size": 1000                          // batch size
  },
  "dest": {
    "index": "accounts-female",
    "op_type": "create",                  // fail instead of overwriting existing ids
    "pipeline": "accounts-ingest"
  },
  "script": {
    "lang": "painless",
    "source": "ctx._source.migrated = true"
  }
}
```

:bulb: `wait_for_completion=false` returns a **task id** immediately — this is the right answer for "reindex a large index without blocking". Then:
```json
GET _tasks/<task_id>
GET _tasks?actions=*reindex&detailed
POST _tasks/<task_id>/_cancel
```

:warning: Gotchas:
- `_reindex` requires `_source` to be **enabled** on the source index.
- `_reindex` does **not** copy settings, mappings, aliases, or the ILM policy — create the destination index (or its template) first.
- Reindexing into a **data stream** requires `"op_type": "create"`.
- A `_reindex` from a remote cluster needs `reindex.remote.whitelist` in `elasticsearch.yml` and a `"remote": { "host": ..., "username": ..., "password": ... }` block in `source`.
- Inside a reindex/UBQ **script**, you write to `ctx._source.field`. Inside an **ingest pipeline** script processor you write to `ctx.field`. Mixing these up is the most common silent failure in this section.

## Part 1
:question: Reindex the `accounts-raw` index into `accounts-2021`.

:question: Then reindex `accounts-2021` into `accounts-female` index where only the female account holders are present.

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-update-by-query.html
> :warning: 
Reindex requires _source to be enabled for all documents in the source index.

:warning: check your templates, to make sure they are not forcing `"_source": { "enabled": false },` as this will break reindexing.

```json
POST _reindex
{
  "source": { "index": "accounts-raw"  },
  "dest":   { "index": "accounts-2021" }
}

GET accounts-2021/_count?filter_path=count

// Output 

{
  "count" : 1000
}
```

reindex into `accounts-female`

:bulb: do the term query first, then once you are happy with the output, convert it into a `_reindex`

```json
POST _reindex
{
  "source": { "index": "accounts-raw",
    "query": {
      "term": {
        "gender.keyword": "F"
      }
    }
  },
  "dest":   { "index": "accounts-female" }
}
```

Check
```json
GET accounts-female/_count?filter_path=count

// Output 

{
  "count" : 493
}
```

Check again
```json
GET /accounts-female/_search?filter_path=*.*.*.gender

// Output 

{
  "hits" : {
    "hits" : [
      {
        "_source" : {
          "gender" : "F"
        }
      },
      {
        "_source" : {
          "gender" : "F"
        }
      },
    ...
```
</details>

## Part 2
:question: Give all female account holders in `accounts-2021` a 25% bonus increase on their balance :)

<details>
  <summary>View Solution (click to reveal)</summary>

Get two example docs
```json
GET /accounts-raw/_search?q=gender:F&size=2
```

Note down those ids and get the balances
```json
GET /accounts-raw/_doc/_mget?filter_path=*.*.balance
{
    "ids" : ["13", "25"]
}

//Output

{
  "docs" : [
    {
      "_source" : {
        "balance" : 32838
      }
    },
    {
      "_source" : {
        "balance" : 40540
      }
    }
  ]
}
```

Update the accounts - take note of the number of `updated` docs
```json
POST accounts-2021/_update_by_query
{
  "script": {
    "source": "ctx._source.balance=ctx._source.balance*1.25",
    "lang": "painless"
  },
  "query": {
    "term": {
      "gender": "F"
    }
  }
}


// Output

{
  "took" : 369,
  "timed_out" : false,
  "total" : 493,
  "updated" : 493,
  "deleted" : 0,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```

Get those same ids again to check the balances have increased
```json
GET /accounts-2021/_doc/_mget?filter_path=*.*.balance
{
    "ids" : ["13", "25"]
}

// Output

{
  "docs" : [
    {
      "_source" : {
        "balance" : 41047.5
      }
    },
    {
      "_source" : {
        "balance" : 50675.0
      }
    }
  ]
}
```

</details>
<hr>

# Define and use an ingest pipeline that satisfies a given set of requirements

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/processors.html

## Processors you should recognise on sight

| Processor | Does |
| --- | --- |
| `set` | set a field to a value (supports `{{{field}}}` mustache templating) |
| `append` | add a value to an array field (creates the array if needed) |
| `remove` | delete a field |
| `rename` | rename a field |
| `convert` | change type: `integer`, `long`, `float`, `double`, `string`, `boolean`, `ip`, `auto` |
| `gsub` | regex replace inside a string field |
| `split` / `join` | string ↔ array |
| `trim`, `lowercase`, `uppercase` | string cleanup |
| `grok` | parse unstructured text into fields with named patterns |
| `dissect` | faster, simpler alternative to grok for fixed-delimiter text |
| `date` | parse a string into `@timestamp` |
| `date_index_name` | route a doc to a date-based index |
| `json` | parse a JSON string into an object |
| `kv` | parse `key=value` pairs |
| `dot_expander` | turn `a.b` into a nested object |
| `script` | anything the above cannot do (Painless) |
| `fail` / `drop` | reject or silently discard a document |
| `pipeline` | call another pipeline |
| `enrich` | join in data from another index (needs an enrich policy) |
| `foreach` | run a processor over every element of an array |

## Options every processor supports

```json
{
  "set": {
    "field": "full_name",
    "value": "{{{firstname}}} {{{lastname}}}",
    "if": "ctx.firstname != null",       // conditional (ctx is READ-ONLY here)
    "ignore_failure": true,              // keep going if this one processor fails
    "tag": "set full_name",              // shows up in _simulate output - name your processors
    "on_failure": [                      // per-processor error handling
      { "set": { "field": "error", "value": "{{{_ingest.on_failure_message}}}" } }
    ]
  }
}
```

And the pipeline itself can have a top-level `on_failure`:
```json
PUT _ingest/pipeline/safe-pipeline
{
  "processors": [ { "convert": { "field": "age", "type": "integer" } } ],
  "on_failure": [
    { "set": { "field": "ingest_error", "value": "{{{_ingest.on_failure_message}}}" } }
  ]
}
```

## The four ways to actually *use* a pipeline

```json
// 1. per-request, on indexing
POST accounts-2021/_doc?pipeline=accounts-ingest
{ "firstname": "Ada", "lastname": "Lovelace" }

// 2. on an existing index, in place
POST accounts-2021/_update_by_query?pipeline=accounts-ingest

// 3. as part of a reindex
POST _reindex
{ "source": { "index": "a" }, "dest": { "index": "b", "pipeline": "accounts-ingest" } }

// 4. as an index default, so every write uses it automatically
PUT accounts-2021/_settings
{
  "index.default_pipeline": "accounts-ingest"
}
```
:bulb: `index.final_pipeline` runs *after* `default_pipeline` and after any `?pipeline=` request parameter — use it for "always stamp this field no matter what".

## Simulating

Two forms — learn both. The first tests a pipeline body you have not saved yet, the second tests one you already created:

```json
POST _ingest/pipeline/_simulate            // inline pipeline
{ "pipeline": { "processors": [ ... ] }, "docs": [ { "_source": { ... } } ] }

POST _ingest/pipeline/accounts-ingest/_simulate?verbose    // saved pipeline, step by step
{ "docs": [ { "_source": { ... } } ] }
```
:bulb: `?verbose` shows the document after **each** processor — invaluable when a five-processor pipeline produces the wrong answer and you need to know which step broke it.

Housekeeping:
```json
GET _ingest/pipeline                 // list all
GET _ingest/pipeline/accounts-ingest
DELETE _ingest/pipeline/accounts-ingest
```

<hr>

:question: Apply a pipeline called `accounts-ingest` to the data in `accounts-2021` with the following requirements:

- Add a `tag` called `pipeline_ingest` to show that the document was ingested via the pipeline 
- Combine the `firstname` and `lastname ` fields into a new field called `full_name`
- Increase the `balance` of holders that are `female` and `39 years old and over` by 5%.


**Hint**: Use the simulate API to test your code

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest-apis.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-source-fields <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-metadata-fields <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-ingest-metadata <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/processors.html   <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/script-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/append-processor.html <br>

> :warning: The value of ctx is read-only in if conditions.

> :question: Thus use a script to do more complex work

:bulb: it is probably quicker now to do this into the web UI.

## Simulate first

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      {
        "append": {
          "field": "tags",
          "value": ["pipeline_ingest"]
        }
      },
      {
        "set": {
          "tag": "set full_name",
          "field": "full_name",
          "value": "{{firstname}} {{lastname}}"
        }
      },
      {
        "script": {
          "tag": "39s and over female bonus",
          "if": """
            if (ctx.age >= 39) { 
              if (ctx.gender=="F") { 
                return true 
              }
            } 
            return false
          """,
          "lang": "painless",
          "source": """
            ctx.balance = ctx.balance*1.05
          """
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "account_number": 10000,
        "balance": 1000000,
        "firstname": "George",
        "lastname": "Cross",
        "age": 92,
        "gender": "M",
        "address": "1 Dog Lane",
        "employer": "Wheatens",
        "email": "george@wheatens.com",
        "city": "London",
        "state": "UK"
      }
    },
    {
      "_source": {
        "account_number": 10001,
        "balance": 1000001,
        "firstname": "Millie",
        "lastname": "Cross",
        "age": 84,
        "gender": "F",
        "address": "1 Dog Lane",
        "employer": "Wheatens",
        "email": "millie@wheatens.com",
        "city": "London",
        "state": "UK"
      }
    }
  ]
}
```

Here you will get a lot of output, make sure it matches what you expect to see.


Now copy the working pipeline

```json
PUT _ingest/pipeline/accounts-ingest
{
  "description" : "pipeline to account ingest",
  "processors": [
      {
        "append": {
          "field": "tags",
          "value": ["pipeline_ingest"]
        }
      },
      {
        "set": {
          "tag": "set full_name",
          "field": "full_name",
          "value": "{{firstname}} {{lastname}}"
        }
      },
      {
        "script": {
          "tag": "39s and over female bonus",
          "if": """
            if (ctx.age >= 39) { 
              if (ctx.gender=="F") { 
                return true 
              }
            } 
            return false
          """,
          "lang": "painless",
          "source": """
            ctx.balance = ctx.balance*1.05
          """
        }
      }
    ]
}
```

Then reindex the data with that new pipeline

```json
POST accounts-2021/_update_by_query?pipeline=accounts-ingest

// Output - check the `updated` field

{
  "took" : 651,
  "timed_out" : false,
  "total" : 1000,
  "updated" : 1000,
  "deleted" : 0,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```

### Check some docs

Pick a doc id

```json
POST accounts-raw/_search?filter_path=*.*._id
{
  "size": 1, 
  "query": { 
    "bool": { 
      "must": [
        { "match": { "gender.keyword":   "F"}}
      ],
      "filter": [ 
        { "range": { "age": { "gte": "39" }}}
      ]
    }
  }
}

// Output 

{
  "hits" : {
    "hits" : [
      {
        "_id" : "25"
      }
    ]
  }
}
```


Original data
```json
GET accounts-raw/_doc/25?filter_path=*.balance,*.firstname,*.lastname

// Output 

{
  "_source" : {
    "balance" : 40540,
    "firstname" : "Virginia",
    "lastname" : "Ayala",
    "age" : 39
  }
}
```

Updated data
```json
GET accounts-2021/_doc/25?filter_path=*.balance,*.full_name,*.tags

// Output 

{
  "_source" : {
    "tags" : [
      "pipeline_ingest"
    ],
    "full_name" : "Virginia Ayala",
    "balance" : 42567.0
  }
}
```

```
40540 * 1.05 = 42567
```

</details>
<hr>

# Define runtime fields to retrieve custom values using Painless scripting

:sparkles: **New objective in 8.15** — it did not exist on the 8.1 list.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-mapping-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-retrieving-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-scripting-painless.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/painless-runtime-fields-context.html

> A runtime field is a field that is evaluated **at query time**. Runtime fields let you add fields to existing documents without reindexing your data, and they consume no index space.

:bulb: The **Searching Data** objective ("write a search that utilizes a runtime field") is the query side; see [Searching_Data.md](Searching_Data.md). This objective is the **definition** side — getting the Painless right.

## Supported runtime field types

`boolean`, `composite`, `date`, `double`, `geo_point`, `ip`, `keyword`, `long`, `lookup`

## The Painless rules for runtime fields

1. You **must** call `emit(...)` to produce a value. A runtime script that returns a value instead of emitting one does nothing.
2. Read the source document with `doc['field']` (doc values, fast) or `params._source['field']` (raw JSON, slower but works on non-doc-value fields).
3. `doc['field'].value` **throws** if the field is missing on a document. Guard it with `doc['field'].size() != 0`, or just `return;` early.
4. You can `emit()` more than once — a runtime field can be multi-valued.
5. For `date` runtime fields, `emit()` takes epoch milliseconds: `emit(doc['t'].value.toInstant().toEpochMilli())`.
6. Use triple-quoted strings `"""..."""` for multi-line scripts so you don't have to escape quotes.

## The three places you can define one

### 1. In the index mapping, at creation time

```json
PUT totality-raw-runtime
{
  "mappings": {
    "runtime": {
      "total_time_seconds": {
        "type": "long",
        "script": {
          "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
        }
      }
    },
    "properties": {
      "totality_seconds": { "type": "long" },
      "totality_minutes": { "type": "long" }
    }
  }
}
```

### 2. Added to an existing index's mapping

This is the headline feature — no reindex, applies to every document already in the index, and takes effect immediately:

```json
PUT totality-raw/_mapping
{
  "runtime": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": """
          if (doc['totality_minutes'].size() == 0 || doc['totality_seconds'].size() == 0) { return; }
          emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value);
        """
      }
    }
  }
}
```

Remove it again just as cheaply — `null` deletes it:
```json
PUT totality-raw/_mapping
{
  "runtime": {
    "total_time_seconds": null
  }
}
```

### 3. In a single search request (`runtime_mappings`)

Scoped to that one request; nothing is persisted.

```json
GET totality-raw/_search
{
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "fields": ["total_time_seconds"],
  "_source": false,
  "size": 5
}
```

:warning: **Retrieve with `fields`, never `_source`.** Runtime fields are not in `_source` and never will be.

<hr>

:question: 1. On the `shakespeare` index, define a runtime field `speaker_type` in the **mapping** that emits `"king"` when the speaker's name starts with `KING`, `"queen"` when it starts with `QUEEN`, and `"other"` otherwise. Then aggregate on it.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT shakespeare/_mapping
{
  "runtime": {
    "speaker_type": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['speaker'].size() == 0) { emit('unknown'); return; }
          String s = doc['speaker'].value;
          if (s.startsWith('KING'))       { emit('king');  }
          else if (s.startsWith('QUEEN')) { emit('queen'); }
          else                            { emit('other'); }
        """
      }
    }
  }
}
```

Use it:
```json
GET shakespeare/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_type": { "terms": { "field": "speaker_type" } }
  }
}
```
</details>
<hr>

:question: 2. Define a runtime field that extracts the **domain** from an `email` keyword field, in a search request only.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET accounts-raw/_search?filter_path=aggregations
{
  "size": 0,
  "runtime_mappings": {
    "email_domain": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['email'].size() == 0) { return; }
          String e = doc['email'].value;
          int at = e.indexOf('@');
          if (at > 0) { emit(e.substring(at + 1)); }
        """
      }
    }
  },
  "aggs": {
    "domains": { "terms": { "field": "email_domain", "size": 10 } }
  }
}
```

:bulb: The `grok`/`dissect` Painless helpers also work in runtime scripts and are far cleaner for log parsing:
```json
"source": "emit(grok('%{IP:client_ip} .*').extract(doc['message'].value)?.client_ip)"
```
</details>
<hr>

:question: 3. Define a `date` runtime field `order_day` that emits the order date truncated to the day, and a `boolean` runtime field `is_weekend`.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET kibana_sample_data_ecommerce/_search
{
  "runtime_mappings": {
    "order_day": {
      "type": "date",
      "script": {
        "source": "emit(doc['order_date'].value.toInstant().toEpochMilli())"
      }
    },
    "is_weekend": {
      "type": "boolean",
      "script": {
        "source": """
          int d = doc['order_date'].value.getDayOfWeekEnum().getValue();
          emit(d == 6 || d == 7);
        """
      }
    }
  },
  "size": 3,
  "fields": [ "order_day", "is_weekend" ],
  "_source": false
}
```
</details>
<hr>

## Also worth knowing: `"dynamic": "runtime"`

Instead of indexing every unknown field, map new fields as runtime fields — searchable, zero index cost:

```json
PUT logs-runtime
{
  "mappings": {
    "dynamic": "runtime",
    "properties": {
      "@timestamp": { "type": "date" }
    }
  }
}
```

## Promoting a runtime field to an indexed field

If a runtime field turns out to be queried constantly, the "make it fast" answer is: add it as a real field in the mapping and `_reindex` (or `_update_by_query`) so it gets indexed. Queries do not change — the indexed field simply shadows the runtime one.

---
---

# :books: BONUS — no longer a listed 8.15 objective

# (Bonus) Configure an index so that it properly maintains the relationships of nested arrays of objects <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/nested.html <br>
:question: 1. Using the below data, create an index with a mapping that allows for relationships to be queried.

```json
PUT henry4_r/_doc/_bulk
{"index":{"_index":"henry4_r","_id":"0"}}
{"name":"KING HENRY IV","relationship":[{"name":"PRINCE HENRY","type":"father"}]}
{"index":{"_index":"henry4_r","_id":"1"}}
{"name":"FALSTAFF","relationship":[{"name":"PRINCE HENRY","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"2"}}
{"name":"HOTSPUR","relationship":[{"name":"PRINCE HENRY","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"3"}}
{"name":"WESTMORELAND","relationship":[{"name":"KING HENRY IV","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"4"}}
{"name":"WESTMORELAND","relationship":[{"name":"KING HENRY IV","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"5"}}
{"name":"EARL OF WORCESTER","relationship":[{"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"6"}}
{"name":"GLENDOWER","relationship":[{"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"7"}}
{"name":"EARL OF DOUGLAS","relationship":[{"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"8"}}
{"name":"MORTIMER","relationship":[{"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"9"}}
{"name":"VERNON","relationship":[{"name":"EARL OF WORCESTER","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"10"}}
{"name":"NORTHUMBERLAND","relationship":[{"name":"HOTSPUR","type":"father"}, {"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"11"}}
{"name":"SIR WALTER BLUNT","relationship":[{"name":"KING HENRY IV","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"12"}}
{"name":"GADSHILL","relationship":[{"name":"PRINCE HENRY","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"13"}}
{"name":"ARCHBISHOP OF YORK","relationship":[{"name":"KING HENRY IV","type":"foe"}]}
{"index":{"_index":"henry4_r","_id":"14"}}
{"name":"POINS","relationship":[{"name":"FALSTAFF","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"15"}}
{"name":"BARDOLPH","relationship":[{"name":"FALSTAFF","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"16"}}
{"name":"PETO","relationship":[{"name":"FALSTAFF","type":"friend"}]}
{"index":{"_index":"henry4_r","_id":"17"}}
{"name":"PRINCE HENRY","relationship":[{"name":"FALSTAFF","type":"friend"}, {"name":"PETO","type":"friend"},{"name":"BARDOLPH","type":"friend"},{"name":"POINS","type":"friend"},{"name":"GADSHILL","type":"friend"}]}
```

<details>
  <summary>View Solution (click to reveal)</summary>

The important part here is to add the `"type": "nested"` so that the data is not flattened when ingested.

So anywhere you have more than one item in the relationship list, it will not be found if you do not use the `nested` term. see docs.

```json
PUT henry4_r
{
  "mappings": {
    "properties": {
      "relationship": {
        "type": "nested" 
      }
    }
  }
}

```
</details>
<hr/>

:question: 2. Then query all people that are a `foe` of `KING HENRY IV`

<details>
  <summary>View Solution (click to reveal)</summary>

You need to be using a `nested` query here as well.  Otherwise it will not work.

```json
GET henry4_r/_search
{
  "query": {
    "nested": {
      "path": "relationship",
      "query": {
        "bool": {
          "must": [
            { "match": { "relationship.name": "KING HENRY IV" }},
            { "match": { "relationship.type":  "foe" }} 
          ]
        }
      }
    }
  }
}
// output

{
  "hits" : {
    "hits" : [
      {
        "_source" : {
          "name" : "EARL OF WORCESTER"
        }
      },
      {
        "_source" : {
          "name" : "GLENDOWER"
        }
      },
      {
        "_source" : {
          "name" : "EARL OF DOUGLAS"
        }
      },
      {
        "_source" : {
          "name" : "MORTIMER"
        }
      },
      {
        "_source" : {
          "name" : "ARCHBISHOP OF YORK"
        }
      },
      {
        "_source" : {
          "name" : "NORTHUMBERLAND"
        }
      },
      {
        "_source" : {
          "name" : "HOTSPUR"
        }
      }
    ]
  }
}
```

</details>
<hr/>

:question: 3. Show all friends of `FALSTAFF`


<details>
  <summary>View Solution (click to reveal)</summary>

There should be four.  This is where the nesting comes into play as `FALSTAFF` himself decribes `PRINCE HENRY` as his only friend. But other people describe `FALSTAFF` as their friend.

```
PRINCE HENRY -> FALSTAFF
FALSTAFF -> PRINCE HENRY
POINS -> FALSTAFF
BARDOLPH -> FALSTAFF
PETO -> FALSTAFF
```

# query

```json
GET henry4_r/_search?filter_path=*.*.*.name
{
  "query": {
    "nested": {
      "path": "relationship",
      "query": {
        "bool": {
          "must": [
            { "match": { "relationship.name": "FALSTAFF" }},
            { "match": { "relationship.type":  "friend" }} 
          ]
        }
      }
    }
  }
}

// output

{
  "hits" : {
    "hits" : [
      {
        "_source" : {
          "name" : "POINS"
        }
      },
      {
        "_source" : {
          "name" : "BARDOLPH"
        }
      },
      {
        "_source" : {
          "name" : "PETO"
        }
      },
      {
        "_source" : {
          "name" : "PRINCE HENRY"
        }
      }
    ]
  }
}
```

</details>
<hr/>
