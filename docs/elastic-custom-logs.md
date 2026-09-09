# Elasticsearch — custom logs

Use this when `message` is not clean JSON: plain text, `key=value`, a CRI prefix, or an application-specific format.

For JSON logs, use [elastic-json-logs.md](./elastic-json-logs.md).

Run these steps in **Kibana → Dev Tools**.

## Step 1 — Get one real log

```json
GET <prefix>-*/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

Copy the exact `message` value. Also note your `ES_INDEX_PREFIX`.

## Step 2 — Ask for a pipeline

Paste this prompt into your AI tool with the real sample. Keep the sample unchanged.

```text
I use Elasticsearch ingest pipelines. Fluent Bit stores each log line in the "message" field.

Here is a real message value from my index:

<paste message here>

Write a PUT _ingest/pipeline/parse_<app>_message request that:
- Parses this log format
- Puts useful fields on the document root
- Uses ignore_failure: true where appropriate
- Removes message after successful parse

Also write PUT _index_template/<prefix>-template with index_patterns ["<prefix>-*"] and default_pipeline "parse_<app>_message".

Do not invent fields that are not in the sample. Keep the pipeline safe for messages that do not match.

App name: <app>   Index prefix: <prefix>
```

The pipeline depends on the actual shape of your log. Do not use a generic example if you already have a real document.

## Step 3 — Run the pipeline

Review the generated request, then run the `PUT _ingest/pipeline/...` request in Dev Tools.

## Step 4 — Test with `_simulate`

Use the same message before attaching the pipeline to your index template:

```json
POST _ingest/pipeline/parse_<app>_message/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "<paste the same sample here>"
      }
    }
  ]
}
```

If the output is wrong, send the pipeline and the `_simulate` result back to the AI tool and ask for a fix. Do not attach it to the index until the simulation looks right.

## Step 5 — Attach the pipeline

Run the generated `PUT _index_template/...` request. It should contain:

```json
"index_patterns": ["<prefix>-*"],
"template": {
  "settings": {
    "default_pipeline": "parse_<app>_message"
  }
}
```

## Step 6 — Verify new documents

```json
GET <prefix>-*/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

Generate new traffic. Existing documents are not reprocessed by the index template.
