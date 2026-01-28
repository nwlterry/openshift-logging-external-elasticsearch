Here is the complete guide on configuring **OpenShift Logging 6.1.8** (part of Red Hat OpenShift Logging 6.1 series) to forward logs to an **external Elasticsearch** instance, along with ways to leverage **Elastic's Kubernetes integration** for better observability in Kibana.

The configuration is based on the official Red Hat documentation for Logging 6.x, which uses the `ClusterLogForwarder` CR (in the `observability.openshift.io/v1` API group) and supports Vector as the collector (recommended over legacy Fluentd).

### Forwarding Logs to External Elasticsearch in OpenShift Logging 6.1.8

#### Prerequisites
- Red Hat OpenShift Logging Operator (version 6.1.x) installed in the `openshift-logging` namespace.
- Cluster administrator access.
- An external Elasticsearch cluster (version 8.x recommended; specify `version: 8` in the output).
- A secret in `openshift-logging` namespace containing credentials and TLS artifacts.

#### Step 1: Create Authentication and TLS Secret
Create an opaque secret with username/password and TLS files:

```bash
oc create secret generic external-es-secret \
  -n openshift-logging \
  --from-literal=username=<es-username> \
  --from-literal=password=<es-password> \
  --from-file=ca-bundle.crt=/path/to/ca.crt \
  --from-file=tls.crt=/path/to/client.crt \
  --from-file=tls.key=/path/to/client.key
```

#### Step 2: Create ClusterLogForwarder CR
This example forwards application and infrastructure logs to external Elasticsearch with dynamic indexing. For audit logs, add `audit` to `inputRefs` and ensure compliance requirements.

```yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  serviceAccount:
    name: collector  # Default; or custom SA with bindings
  outputs:
    - name: external-elasticsearch
      type: elasticsearch
      elasticsearch:
        url: https://your-es-cluster.example.com:9200
        version: 8  # Required for ES 8.x+
        index: '{.log_type}-write'  # e.g., app-write, infra-write, audit-write
        authentication:
          username:
            key: username
            secretName: external-es-secret
          password:
            key: password
            secretName: external-es-secret
        tls:
          ca:
            key: ca-bundle.crt
            secretName: external-es-secret
          certificate:
            key: tls.crt
            secretName: external-es-secret
          key:
            key: tls.key
            secretName: external-es-secret
  pipelines:
    - name: forward-to-external
      inputRefs:
        - application
        - infrastructure
        # - audit   # Add if needed for audit logs
      outputRefs:
        - external-elasticsearch
```

Apply with:
```bash
oc apply -f clusterlogforwarder.yaml
```

- **Key notes**:
  - If no `ClusterLogForwarder` exists, logs go to the internal store (if configured in `ClusterLogging` CR).
  - With a `ClusterLogForwarder`, default forwarding to internal ES stops unless you add a pipeline with the default output.
  - For ES 8.x, `version: 8` is mandatory (legacy behavior assumes v6).
  - Use `index` templates like `{.log_type||"undefined"}-write` to match Elastic's expected patterns.

#### Step 3: Grant Permissions (if using custom SA)
```bash
oc adm policy add-cluster-role-to-user collect-application-logs system:serviceaccount:openshift-logging:collector
oc adm policy add-cluster-role-to-user collect-infrastructure-logs system:serviceaccount:openshift-logging:collector
# For audit:
oc adm policy add-cluster-role-to-user collect-audit-logs system:serviceaccount:openshift-logging:collector
```

#### Step 4: Verify
- Check status: `oc get clusterlogforwarder instance -n openshift-logging -o yaml`
- View collector pods: `oc get pods -n openshift-logging | grep collector`
- Query your external ES for indices like `app-write-*`, `infra-write-*`.

### Leveraging Elastic Kubernetes Integration

Elastic provides a dedicated **Kubernetes integration** in Elastic Stack (via Fleet/Integrations in Kibana), which includes:
- Pre-built index templates and ingest pipelines for ECS-formatted logs (`logs-kubernetes.*` data streams).
- Dashboards for pod/container logs, events, audit logs, metrics, etc.

Since OpenShift restricts privileged host access (e.g., no direct Elastic Agent daemonset without SCC adjustments), the best approach is:

#### Recommended: Use Ingest Pipelines to Normalize OpenShift Logs
1. **In Kibana → Integrations** — Install the **Kubernetes** integration (adds assets like pipelines for `logs-kubernetes.container_logs`, `logs-kubernetes.audit_logs`).
2. **Create Custom Ingest Pipelines** in Elasticsearch to transform OpenShift logs:
   - Rename fields: e.g., `kubernetes.pod_name` → `kubernetes.pod.name`, `kubernetes.container_name` → `kubernetes.container.name`.
   - Sanitize annotations/labels (replace dots with underscores).
   - Add ECS fields like `service.name`, `host.name`.
   - Reroute from OpenShift indices (`app-write-*`) to Elastic data streams (`logs-kubernetes.container_logs-*`).

   Example pipeline snippet (add via Kibana Dev Tools):
   ```json
   {
     "description": "Normalize OpenShift to ECS",
     "processors": [
       { "rename": { "field": "kubernetes.pod_name", "target_field": "kubernetes.pod.name" } },
       { "set": { "field": "service.name", "value": "{{kubernetes.labels.app}}" } }
     ]
   }
   ```

3. **Set Default Pipeline** on your write indices (e.g., `app-write-*`) to use the normalization pipeline.
4. **Forward from OpenShift** using the `ClusterLogForwarder` above — logs arrive in temporary indices, get processed, and appear in Kubernetes dashboards.

This avoids deploying Elastic Agent/Beats in OpenShift (due to SCC and host mount issues) while gaining full integration benefits.

#### Alternative: Deploy Elastic Stack Natively on OpenShift (ECK)
If you can run Elasticsearch inside OpenShift (or want hybrid):
- Install **Elastic Cloud on Kubernetes (ECK) Operator** via OperatorHub.
- Deploy Elasticsearch + Kibana CRs.
- Use Metricbeat/Filebeat (with privileged SCC) for metrics/events.
- Forward OpenShift logs via `ClusterLogForwarder` to this internal ES.
- Full Kubernetes integration dashboards become available out-of-the-box.

This provides deeper integration but requires managing storage and scaling within OpenShift.

For production, monitor index mappings, test TLS, and tune retention/ILM policies in Elasticsearch. Refer to Red Hat docs for 6.1.x for any patch-specific notes (no major forwarding changes in 6.1.8).
