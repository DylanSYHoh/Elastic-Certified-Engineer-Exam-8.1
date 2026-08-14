# Attempts to provide some clarity on concepts and terms that are found on the exam.
This isn't in any particular order. Aligned to the **8.15** objectives.

## :sparkles: Runtime Field
A field that is **evaluated at query time** rather than at index time. It is defined by a Painless script that calls `emit(...)`, it consumes no index space, and it can be added to or removed from an existing index instantly — no reindex required.

Three places to define one: in the index mapping at creation, added to an existing mapping (`PUT idx/_mapping` with a `runtime` block), or inline in a single search request under `runtime_mappings`.

:warning: A runtime field is **never** in `_source`. Retrieve it with the `fields` parameter of the search request. Related: `"dynamic": "runtime"` maps *unknown* fields as runtime fields instead of indexing them.

## :sparkles: Snapshot Lifecycle Management (SLM)
Automates taking and deleting snapshots on a schedule, so you don't have to script `PUT _snapshot/repo/snap` in cron. A policy has a `schedule` (cron, **with a seconds field**), a `name` (supports date math), a `repository` (which must already exist), a `config` (the same body you would send to the snapshot API), and optional `retention` rules (`expire_after`, `min_count`, `max_count`).

Retention is applied by a separate periodic task, not at snapshot time. `POST _slm/policy/<id>/_execute` runs a policy immediately, which is how you test one without waiting for the schedule.

## Painless
Elasticsearch's built-in scripting language — a safe, sandboxed, Java-like syntax. Where it shows up on the exam:

| Context | You write to | Notes |
| --- | --- | --- |
| Runtime field | `emit(value)` | reads via `doc['f'].value` or `params._source['f']` |
| Ingest pipeline `script` processor | `ctx.field` | note: **no** `_source` |
| `_update` / `_update_by_query` / `_reindex` script | `ctx._source.field` | note: **with** `_source` |
| Script query / script sort | `return value` | reads via `doc['f'].value` |

The `ctx` vs `ctx._source` distinction is the single most common silent mistake in this material.

## Component Template vs Index Template
A **component template** is a reusable fragment of settings/mappings/aliases. It does nothing on its own.

A **composable index template** (`PUT _index_template/...`) is what actually matches an index pattern. It can pull in component templates via `composed_of`, and its own inline `template` block overrides them.

When several index templates match the same new index, the highest `priority` wins **outright** — they are not merged with each other. Component templates listed in `composed_of` *are* merged, in array order.

:warning: The legacy `PUT _template/...` API is deprecated in 8.x and removed in 9.x. Use `_index_template`.

## Data Tiers
`data_content`, `data_hot`, `data_warm`, `data_cold`, `data_frozen` — node roles that ILM's `migrate` action moves indices between as they age. The frozen tier is where searchable snapshots with `shared_cache` storage live.

## Point In Time (PIT)
A lightweight, named view of the index state at a moment in time (`POST /idx/_pit?keep_alive=5m`). Combined with `search_after`, it is the modern replacement for `scroll` when you need to page past 10,000 results consistently. It also unlocks the `_shard_doc` tiebreaker sort field.

## Searchable Snapshot
A snapshot that has been **mounted** as a searchable index rather than fully restored. Storage is either `full_copy` (cold tier — a local copy, no replicas needed) or `shared_cache` (frozen tier — only a partial cache locally, data fetched from the repository on demand). Cuts storage cost dramatically for read-only data.

## Health API (`GET _health_report`)
Added in 8.7. Rather than just a green/yellow/red status, it returns per-indicator diagnoses (`shards_availability`, `disk`, `ilm`, `slm`, `master_is_stable`, …) with the **cause**, a suggested **action**, and a doc link. For the "diagnose shard issues" objective, run this before `_cluster/allocation/explain`.



## Sub-Aggregations
These allow you to embed aggregations inside other aggregations.<br>
Therefore, sub-aggregations allow continuously refine and separate groups of criteria of interest, then apply metrics at various levels in the aggregation hierarchy to generate your report.
<br>
## Bucket vs Metric Aggregations
The difference is that metric aggregations take the data and aggregate some extracted values. Bucket aggregations on the other hand just bucket data according to some criteria.
<br>
Metric Aggregations:<br>
The aggregations in this family compute metrics based on values extracted in one way or another from the documents that are being aggregated. The values are typically extracted from the fields of the document (using the field data), but can also be generated using scripts.

Bucket Aggregations:<br>
Bucket aggregations don’t calculate metrics over fields like the metrics aggregations do, but instead, they create buckets of documents. Each bucket is associated with a criterion (depending on the aggregation type) which determines whether or not a document in the current context "falls" into it. In other words, the buckets effectively define document sets. In addition to the buckets themselves, the bucket aggregations also compute and return the number of documents that "fell into" each bucket.

## Index and Index Template
What is an index and index template? What is the difference between the 2, when would use 1 vs the other, and benefits of 1 vs the other!

## Dynamic Template
These allow you greater control of how Elasticsearch maps your data beyond the default dynamic field mapping rules. You enable dynamic mapping by setting the dynamic parameter to true or runtime. You can then use dynamic templates to define custom mappings that can be applied to dynamically added fields based on the matching condition

## Index Lifecycle Management
You can configure index lifecycle management (ILM) policies to automatically manage indices according to your performance, resiliency, and retention requirements.

## Analyzer
Video on subject to help with understanding: https://www.youtube.com/watch?v=pEV4yNlD5wI <br>
For more information about what makes up an analyzer, check out this article on the anatomy of an analyzer: https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html <br>
Analyzers perform the text analysis for Elasticsearch. Analyzers are a set of rules that govern the entire process. <br>
If you want to tailor your search experience, you can choose a different built-in analyzer or even configure a custom one. A custom analyzer gives you control over each step of the analysis process, including: <br>

- Changes to the text before tokenization <br>
- How text is converted to tokens <br>
- Normalization changes made to tokens before indexing or search <br> 

## Data Stream
A data stream lets you store append-only time series data across multiple indices while giving you a single named resource for requests. Data streams are well-suited for logs, events, metrics, and other continuously generated data.

## Mapping 
Mapping is the process of defining how a document, and the fields it contains, are stored and indexed. <br>
Each document is a collection of fields, which each have their own data type. When mapping your data, you create a mapping definition, which contains a list of fields that are pertinent to the document. <br>
A mapping definition also includes metadata fields, like the _source field, which customize how a document’s associated metadata is handled. <br>

## Multi-Fields
It is often useful to index the same field in different ways for different purposes. For instance, a string field could be mapped as a text field for full-text search, and as a keyword field for sorting or aggregations. Alternatively, you could index a text field with the standard analyzer, the english analyzer, and the french analyzer.
<br>
This is the purpose of multi-fields. Most field types support multi-fields via the fields parameter.

## Ingest Pipeline
Ingest pipelines let you perform common transformations on your data before indexing.  
<br>
A pipeline consists of a series of configurable tasks called processors. Each processor runs sequentially, making specific changes to incoming documents. After the processors have run, Elasticsearch adds the transformed documents to your data stream or index.

## Pagination
Pagination allows you to break up the search results into more manageable chunks.<br> A great way to think of this is that if you do a search engine search with lets say 10 Million results, the search engine will break those results into chunks of 20, before you need to go to the next page. That's pagination.

## Index Aliases
An index alias is a secondary name for one or more indices. Most Elasticsearch APIs accept an alias in place of a data stream or index name.
<br>
You can change the data streams or indices of an alias at any time. If you use aliases in your application’s Elasticsearch requests, you can reindex data with no downtime or changes to your app’s code.

## Pagination methods, compared
| Method | Good for | Ceiling |
| --- | --- | --- |
| `from` + `size` | page-numbered UIs | `from + size` ≤ `index.max_result_window` (10,000) |
| `search_after` + PIT | deep paging, exporting everything | none; needs a unique tiebreaker sort |
| `scroll` | legacy bulk export | deprecated in 8.x |
| composite agg + `after_key` | paging **buckets**, not documents | n/a |

## Search Templates
A search template is a stored search you can run with different variables. *(No longer an 8.15 objective.)*

## Term Query
Query the data based upon a term. 

## Phrase Query
Query the data based upon a phrase. 

## Asynchronous Search
Asynchronous search lets you submit a search request that gets executed asynchronously, monitor the progress of the request, and retrieve results at a later stage. You can also retrieve partial results as they become available but before the search has completed.

## Cross Cluster Search
Cross-cluster search lets you run a single search request against one or more remote clusters. For example, you can use a cross-cluster search to filter and analyze log data stored on clusters in different data centers.

## Cross Cluster Replication
Using an active-passive model, cross-cluster replication, you can replicate indices across clusters to for example:<br>
- Continue handling search requests in the event of a datacenter outage <br>
- Prevent search volume from impacting indexing throughput <br>
- Reduce search latency by processing search requests in geo-proximity to the user<br>
 
 
