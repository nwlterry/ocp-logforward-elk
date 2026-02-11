Here is a practical guide to setting up **log forwarding** in **OpenShift 4.18**, including filtering, based on the current Red Hat OpenShift Logging implementation (typically Logging 5.8+ for 4.18, using the modern **ClusterLogForwarder** API with Vector collector).

The official documentation for OpenShift 4.18 Logging is available here:  
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/logging/index

Key sections include log collection/forwarding and configuring the ClusterLogForwarder.

### Prerequisites
- The **Red Hat OpenShift Logging Operator** must be installed (usually version 5.8 or later for OCP 4.18).
- A default **ClusterLogging** custom resource (CR) named `instance` usually exists in the `openshift-logging` namespace (even if you're forwarding externally and not using the internal store).
- You have cluster-admin privileges.
- Decide on your **output** destination (e.g. LokiStack, external Elasticsearch, CloudWatch, Splunk, Kafka, syslog, etc.).

### Step 1: Basic Log Forwarding Setup (No Filtering)
Create or edit a **ClusterLogForwarder** CR in the `openshift-logging` namespace.

**Example: Forward all logs to an external HTTPS endpoint (e.g. Loki or Elasticsearch)**

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: external-loki
      type: lokistack   # or elasticsearch, cloudwatch, kafka, syslog, etc.
      url: https://lokistack-ingester.openshift-logging.svc:3100
      secret:
        name: lokistack-secret   # contains TLS certs / credentials if needed

  pipelines:
    - name: forward-all
      inputRefs:
        - application
        - infrastructure
        - audit
      outputRefs:
        - external-loki
        # - default   # optional: keep sending to internal store too
```

Apply it:

```bash
oc apply -f clf-forward-all.yaml
```

This forwards **application** (container/pod logs), **infrastructure** (node/journal logs), and **audit** (API server / kube events) logs.

### Step 2: Adding Filtering
Filtering is very flexible in recent versions. You can filter at different levels:

- **By log source** (namespace, labels, pod name) — most common for application logs
- **By content** (drop logs matching regex, prune fields)
- **By severity / facility** (especially for audit logs)

#### Option A: Filter by namespace / labels (application logs)

```yaml
spec:
  inputs:
    - name: my-important-apps
      application:
        selector:
          matchLabels:
            app.kubernetes.io/part-of: critical-service
        namespaces:
          - prod-frontend
          - prod-backend
          - monitoring

    - name: noisy-namespace
      application:
        namespaces:
          - dev-sandbox

  pipelines:
    - name: important-to-loki
      inputRefs:
        - my-important-apps
      outputRefs:
        - external-loki

    - name: noisy-to-splunk   # example different destination
      inputRefs:
        - noisy-namespace
      outputRefs:
        - splunk-output
```

This collects only logs from specific namespaces or pods with certain labels.

You can also **exclude** namespaces:

```yaml
application:
  namespaces:
    - "!logging"
    - "!openshift-*"
```

#### Option B: Content-based filtering (drop / prune logs)

Use filters to drop unwanted records or prune fields.

```yaml
spec:
  filters:
    - name: drop-debug-and-info
      type: drop
      drop:
        - condition: '.level == "debug" or .level == "info"'

    - name: drop-health-checks
      type: drop
      drop:
        - condition: '.message | test("healthz|readyz|liveness")'

    - name: prune-sensitive
      type: prune
      prune:
        - in: ['.kubernetes.pod_labels.secret', '.kubernetes.annotations.token']

  pipelines:
    - name: cleaned-logs
      inputRefs:
        - application
      filterRefs:
        - drop-debug-and-info
        - drop-health-checks
        - prune-sensitive
      outputRefs:
        - external-loki
```

#### Option C: Audit log filtering (very useful to reduce volume)

Audit logs can be extremely verbose. Filter by API group, verb, user, etc.

```yaml
spec:
  inputs:
    - name: filtered-audit
      audit:
        filter:
          - type: drop
            condition: '.verb == "get" and .user.username == "system:serviceaccount:kube-system:default"'
          - type: drop
            condition: '.requestURI | test("/healthz|/metrics")'

  pipelines:
    - name: audit-lean
      inputRefs:
        - filtered-audit
      outputRefs:
        - external-loki
```

### Step 3: Common Full Example (with Filtering)

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: loki-prod
      type: lokistack
      url: https://lokistack-ingester.openshift-logging.svc:3100
      secret:
        name: lokistack-gateway-secret

  inputs:
    - name: prod-apps-only
      application:
        namespaces:
          - production
          - finance
        selector:
          matchLabels:
            critical: "true"

  filters:
    - name: drop-low-level
      type: drop
      drop:
        - condition: '.level in ["trace", "debug"]'

  pipelines:
    - name: prod-pipeline
      inputRefs:
        - prod-apps-only
        - infrastructure   # nodes, journal
      filterRefs:
        - drop-low-level
      outputRefs:
        - loki-prod

    - name: audit-pipeline
      inputRefs:
        - audit
      outputRefs:
        - loki-prod   # or separate secure output
```

### Step 4: Verify & Troubleshoot
- Check status: `oc get clusterlogforwarder instance -n openshift-logging -o yaml`
- Collector pods: `oc get pods -n openshift-logging | grep collector`
- Logs: `oc logs -n openshift-logging -l app.kubernetes.io/component=collector`
- Test forwarding: Create a pod in a matching namespace and generate logs (`echo "test log" >&2`).

For the most accurate and up-to-date examples (including new filter syntax or output types), always refer to the **4.18 Logging** docs:

- https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/logging/log-collection-and-forwarding
- https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/logging/configuring-log-forwarding

If you're targeting a specific output (Splunk, CloudWatch, etc.) or have a particular filtering use case, let me know for a more tailored example!
