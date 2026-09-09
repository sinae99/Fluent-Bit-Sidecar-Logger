# Fluent Bit sidecar logger

Send logs from one Kubernetes Deployment to Elasticsearch and view them in Kibana.

No DaemonSet, No cluster-wide RBAC + logging agent


## how

The app writes logs to stdout. 

Kubernetes writes them to the node log directory.

Fluent Bit reads only this app's log files and sends them to Elasticsearch.


```text
app stdout
    │
    ▼
/var/log/containers/*_<namespace>_<container>-*.log
    │
    ▼
Fluent Bit sidecar
    │
    ▼
Elasticsearch  →  ingest pipeline  →  Kibana
```



## files

```text
github/
├── README.md
├── config/
├── docs/
└── examples/

```

`examples/` is a complete sample.

`config/fluent-bit.conf` is a reusable Fluent Bit config template when you want to merge it into an existing Deployment.

## start

### 1. set the Elastic Secret

Copy `examples/secret-elasticsearch.yaml` and replace the placeholders.

```yaml
stringData:
  ES_HOST: "your-elasticsearch-host"
  ES_PORT: "443"
  ES_USER: "your-username"
  ES_PASSWORD: "your-password"
  ES_INDEX_PREFIX: "your-app-logs"
  ES_TLS: "On"
```


### 2. log file path

Open `examples/cm-fluentbit.yaml` and update the `[INPUT]` `Path`.

Kubernetes names container log files like this:

```text
<pod-name>_<namespace>_<container-name>-<id>.log
```

For an app named `api` in the `default` namespace:

```ini
Path /var/log/containers/*_default_api-*.log
```

Verify this on node.

### 3. Apply the manifests

```bash
kubectl apply -f examples/secret-elasticsearch.yaml -n <namespace>
kubectl apply -f examples/cm-fluentbit.yaml -n <namespace>
kubectl apply -f examples/deployment.yaml -n <namespace>
```

The sample Deployment contains an `nginx` app and the Fluent Bit sidecar.


```bash
kubectl apply -k examples/
```

Update the namespace and the `Path` first.

### 4. Check the sidecar

```bash
kubectl logs <pod-name> -c fluent-bit --tail=100 -f -n <namespace>
```
```
kubectl exec -it <pod-name> -c fluent-bit -n <namespace> -- \
  ls /var/log/containers/ | grep <container-name>
```

If the file list is empty, fix the `Path` before checking Elasticsearch.

### 5. Kibana

In **Stack Management → Data Views**, create:

```text
<ES_INDEX_PREFIX>-*
```

Use `@timestamp` as the time field. Open **Discover**, select the data view, and wait for a new log record.


---
## If your logs need parsing

### Elasticsearch logs

Fluent Bit sends the raw application line in `message`. If the app writes JSON, `message` looks like this:

```json
{"remote_ip":"1.2.3.4","method":"GET","uri":"/health","status":200,"latency":12345678}
```

Use [docs/elastic-json-logs.md](docs/elastic-json-logs.md) to parse it and put the fields at the document root.

---

If the message is plain text, key/value text, or has a custom format, use [docs/elastic-custom-logs.md](docs/elastic-custom-logs.md). 

It includes the prompt used to generate a pipeline from a real Elasticsearch sample.
