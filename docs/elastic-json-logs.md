# Elasticsearch — JSON logs

When `message` is JSON:

```json
{"remote_ip":"1.2.3.4","method":"GET","uri":"/health","status":200,"latency":12345678,"latency_human":"12.3ms"}
```

Fluent Bit stores this whole line in `message`. Run the following in **Kibana → Dev Tools**.

One generic pipeline promotes all top-level JSON fields. You do not need to list every field by hand.

## Step 1 — Create the pipeline

```json
PUT _ingest/pipeline/parse_json_message
{
  "description": "Parse JSON in message and promote all fields to document root",
  "processors": [
    {
      "json": {
        "field": "message",
        "add_to_root": true,
        "ignore_failure": true
      }
    },
    {
      "remove": {
        "field": "message",
        "ignore_failure": true,
        "if": "ctx.message != null && ctx.message instanceof String && ctx.message.startsWith('{')"
      }
    }
  ]
}
```

## Step 2 — Test with `_simulate`

This does not index anything. Replace the sample with the real `message` from your index.

```json
POST _ingest/pipeline/parse_json_message/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "{\"remote_ip\":\"1.2.3.4\",\"method\":\"GET\",\"uri\":\"/health\",\"status\":200,\"latency\":12345678}"
      }
    }
  ]
}
```

The fields should appear at the root of `doc._source`.

## Step 3 — Attach it to your index

Replace `api-logs` with your `ES_INDEX_PREFIX`.

```json
PUT _index_template/api-logs-template
{
  "index_patterns": ["api-logs-*"],
  "template": {
    "settings": {
      "default_pipeline": "parse_json_message"
    }
  }
}
```

New documents in matching indices will use the pipeline. Existing documents are not changed.

## Step 4 — Verify

```json
GET api-logs-*/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

You should now see fields such as `method`, `uri`, `status`, `remote_ip`, and `latency` at the document root.

For a non-JSON message, use [elastic-custom-logs.md](./elastic-custom-logs.md).
