# Elasticsearch Index Size Tests

Supporting files for testing Elasticsearch index sizes. The set contains index template files for Elasticsearch version 2.x/5.x and 6.x. An equivalent version of Logstash should be needed to load the testing data. 

## Files

- `index-size-tests-logstash.conf`: Logstash config
- `v2-*.json`: Index templates for Elasticsearch 2.x and 5.x
- `v6-*.json`: Index templates for Elasticsearch 6.x
- `logs.gz`: 300,000 lines of access logs of elastic.co web site, 67,644,119 bytes on the disk

## Testing steps

**Ingesting logs**

Uncompress `logs.gz` with `gunzip logs.gz` command. Chose one of an appropriate index template from `v6-*.json` files. E.g. `v6-index-size-tests-keyword-default.json` applies the `keyword` type to all the string fields and the `default` compression wil be used. `v6-index-size-tests-keyword-retain-message-text-best_compression.json` applies the `keyword` type to all the string fields other than retained `message` field which has the `text` type and the `best_compression` will be defined.

Carefully look at the Elasticsearch output section of `index-size-tests-logstash.conf` and modify as necessary.

Run the command like below to ingest logs.

```
cat logs | INDEX=v6-index-size-tests-keyword-default VAR=_omit_message logstash -f index-size-tests-logstash.conf
cat logs | INDEX=v6-index-size-tests-keyword-retain-message-text-best_compression logstash -f index-size-tests-logstash.conf
```

The `INDEX` environment variable must take the index name and the index template name. Once `_omit_message` is defined as the `VAR`, the `message` field generated by Logstash which contains the whole log line will be omit before ingesting.

**Optimizing the index**

Optimize the index to one segment by the `_forcemerge` API.

```
POST v6-index-size-tests-keyword-default/_forcemerge?max_num_segments=1
```

**Getting the size of the index**

Get the size on the disk by the `_stat` API as below and find out the number under the `primeries.store.size_in_bytes`.

```
GET v6-index-size-tests-keyword-default/_stats
```

## Testing results

Here are some testing results run upon Elasticsearch version 6.x. 

| Strategy         | Explicit mappings        | `default`          | `best_compression` |
|:-----------------|:-------------------------|:------------------:|:------------------:|
| Minimal size     | keyword                  | 41,154,570 (60.8%) | 30,422,525 (45.0%) |
| Production       | keyword, ip for clientip | 43,001,634 (63.6%) | 31,948,559 (47.2%) | 
| V5 default       | keyword + text           | 51,574,003 (76.2%) | 40,167,806 (59.4%) |
| Full text search | keyword, text on message | 69,989,301 (103.5%)| 51,782,277 (76.6%) |
