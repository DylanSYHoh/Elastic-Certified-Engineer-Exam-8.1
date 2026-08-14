# Documentation References — Elastic Certified Engineer 8.15

All links point at the **8.15** documentation, which is the version served during the exam.

:bulb: In the exam you only get https://www.elastic.co/guide/index.html — no Google, no Stack Overflow. Practise finding these pages by navigating the docs sidebar and using the built-in search, not by following the links below.

---

## 1. Data Management
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-management.html

**Define an index that satisfies a given set of requirements** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-create-index.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-modules.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-update-settings.html

**Define and use a dynamic template that satisfies a given set of requirements** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-templates.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-mapping.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-field-mapping.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic.html

**Define an Index Lifecycle Management policy for a time-series index** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-lifecycle-management.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-up-lifecycle-policy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-actions.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-rollover.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-index-lifecycle.html (phases & `min_age`) <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-explain-lifecycle.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-tiers.html

**Define an index template that creates a new data stream** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-streams.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-up-a-data-stream.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/use-a-data-stream.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-streams-change-mappings-and-settings.html

**Background: index templates (prerequisite, not a listed objective)** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-templates.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-component-template.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-simulate-index.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-simulate-template.html

---

## 2. Searching Data

**Write and execute a search query for terms and/or phrases in one or more fields of an index** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/full-text-queries.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-match-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-match-query-phrase.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-multi-match-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-query-string-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/term-level-queries.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-term-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-terms-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-range-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-wildcard-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-regexp-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-exists-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-fuzzy-query.html

**Write and execute a search query that is a Boolean combination of multiple queries and filters** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/compound-queries.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-bool-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-filter-context.html

**Write an asynchronous search** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/async-search.html

**Write and execute metric and bucket aggregations** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-metrics.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-datehistogram-aggregation.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-terms-aggregation.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations.html#return-only-agg-results

**Write and execute aggregations that contain sub-aggregations** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations.html#run-sub-aggs <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-pipeline.html

**Write and execute a query that searches across multiple clusters** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-cross-cluster-search.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-search.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-resolve-cluster-api.html

**Write and execute a search that utilizes a runtime field** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-search-request.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-retrieving-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-fields.html

---

## 3. Developing Search Applications

**Sort the results of a query by a given set of requirements** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/sort-search-results.html

**Implement pagination of the results of a search query** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/paginate-search-results.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/point-in-time-api.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-composite-aggregation.html

**Define and use index aliases** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/aliases.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-aliases.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-get-alias.html

---

## 4. Data Processing

**Define a mapping that satisfies a given set of requirements** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-types.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-params.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/explicit-mapping.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/doc-values.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-source-field.html

**Define and use multi-fields with different data types and/or analyzers** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/multi-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analyzer-anatomy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-custom-analyzer.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-normalizers.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-analyze.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/specify-analyzer.html

**Use the Reindex API and Update By Query API to reindex and/or update documents** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-reindex.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-update-by-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-delete-by-query.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/tasks.html

**Define and use an ingest pipeline that satisfies a given set of requirements** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/processors.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/simulate-pipeline-api.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/handling-failure-in-pipelines.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/script-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/append-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/grok-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dissect-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/date-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/convert-processor.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-source-fields <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-metadata-fields <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest.html#access-ingest-metadata

**Define runtime fields to retrieve custom values using Painless scripting** :sparkles: *(new in 8.15)* <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-mapping-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-retrieving-fields.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-indexed.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-scripting-painless.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/painless-runtime-fields-context.html <br>
https://www.elastic.co/guide/en/elasticsearch/painless/8.15/painless-api-reference.html (the Painless API reference — bookmark this one)

---

## 5. Cluster Management

**Diagnose shard issues and repair a cluster's health** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/health-api.html :sparkles: *(start here)* <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-health.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-health.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-indices.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-shards.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-allocation.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-allocation-explain.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-reroute.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-cluster.html (disk watermarks, allocation deciders) <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/red-yellow-cluster-status.html

**Backup and restore a cluster and/or specific indices** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshot-restore.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-register-repository.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-restore-snapshot.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/restore-snapshot-api.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/api-conventions.html#api-date-math-index-names <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-backup.html

**Configure a snapshot to be searchable** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots-api-mount-snapshot.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-searchable-snapshot.html

**Configure a cluster for cross-cluster search** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-cross-cluster-search.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/remote-clusters.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/remote-clusters-settings.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-remote-info.html

**Implement cross-cluster replication** <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/xpack-ccr.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-getting-started-tutorial.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-apis.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-put-follow.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-put-auto-follow-pattern.html

**Automate snapshots with Snapshot Lifecycle Management** :sparkles: *(new in 8.15)* <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html#automate-snapshots-slm <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/getting-started-snapshot-lifecycle-management.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/slm-api-put-policy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/slm-api-execute-lifecycle.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/slm-api-get-policy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/slm-api-get-stats.html

---

## :books: Bonus — dropped from the objective list, still useful

Index templates for a given pattern <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-templates.html

Highlighting <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/highlighting.html

Search templates <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-template.html

Nested field types and queries <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/nested.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-nested-query.html

Role-based access control <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/built-in-roles.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-privileges.html

---

## Generally useful

REST API conventions (date math, multi-index, `filter_path`) <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/api-conventions.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/common-options.html

Full API index — the fastest way to find a syntax you half-remember <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/rest-apis.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat.html
