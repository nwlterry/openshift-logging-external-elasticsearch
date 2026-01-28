# Configuring OpenShift Logging 6.1.8 to Forward Logs to External Elasticsearch

OpenShift Logging 6.1.8 (part of Red Hat OpenShift Logging for OpenShift 4.15+) uses the ClusterLogForwarder to route logs to external systems like Elasticsearch. This guide incorporates details from the Elastic Observability Labs blog on collecting OpenShift container logs with the Red Hat Logging Operator, focusing on ECS normalization for Kubernetes integration compatibility. The blog is based on OpenShift 4.14, but the configurations apply similarly to 6.1.8 with Vector as the default collector.

#### Prerequisites
- OpenShift cluster (4.15+ for Logging 6.1.8).
- Red Hat OpenShift Logging Operator installed in the `openshift-logging` namespace.
- External Elasticsearch 8.x cluster.
- Kubernetes integration assets installed in Elasticsearch (index templates and ingest pipelines for `logs-kubernetes.container_logs` and `logs-kubernetes.audit_logs`).
- Administrator access.
- Network connectivity from OpenShift to Elasticsearch over HTTPS.

#### Step 1: Install Kubernetes Integration Assets in Elasticsearch
Install Elastic's Kubernetes integration to enable ECS-compliant mappings and pipelines. This can be done via Fleet or manually.

Refer to: [Elastic Guide on Installing Integration Assets](https://www.elastic.co/guide/en/fleet/8.11/install-uninstall-integration-assets.html#install-integration-assets).

#### Step 2: Create Ingest Pipelines for ECS Normalization
These pipelines transform OpenShift logs to match ECS and Kubernetes integration formats, handling field renames and sanitizing labels/annotations.

**Container Logs Pipeline (`openshift-2-ecs`)**:
```json
PUT _ingest/pipeline/openshift-2-ecs
{
  "processors": [
    { "rename": { "field": "kubernetes.pod_id", "target_field": "kubernetes.pod.uid", "ignore_missing": true } },
    { "rename": { "field": "kubernetes.pod_ip", "target_field": "kubernetes.pod.ip", "ignore_missing": true } },
    { "rename": { "field": "kubernetes.pod_name", "target_field": "kubernetes.pod.name", "ignore_missing": true } },
    { "rename": { "field": "kubernetes.namespace_name", "target_field": "kubernetes.namespace", "ignore_missing": true } },
    { "rename": { "field": "kubernetes.namespace_id", "target_field": "kubernetes.namespace_uid", "ignore_missing": true } },
    { "rename": { "field": "kubernetes.container_id", "target_field": "container.id", "ignore_missing": true } },
    {
      "dissect": {
        "field": "container.id",
        "pattern": "%{container.runtime}://%{container.id}",
        "ignore_failure": true
      }
    },
    { "rename": { "field": "kubernetes.container_image", "target_field": "container.image.name", "ignore_missing": true } },
    { "set": { "field": "kubernetes.container.image", "copy_from": "container.image.name", "ignore_failure": true } },
    { "set": { "field": "container.name", "copy_from": "kubernetes.container_name", "ignore_failure": true } },
    { "rename": { "field": "kubernetes.container_name", "target_field": "kubernetes.container.name", "ignore_missing": true } },
    { "set": { "field": "kubernetes.node.name", "copy_from": "hostname", "ignore_failure": true } },
    { "rename": { "field": "hostname", "target_field": "host.name", "ignore_missing": true } },
    { "rename": { "field": "level", "target_field": "log.level", "ignore_missing": true } },
    { "rename": { "field": "file", "target_field": "log.file.path", "ignore_missing": true } },
    { "set": { "field": "orchestrator.cluster.name", "copy_from": "openshift.cluster_id", "ignore_failure": true } },
    {
      "dissect": {
        "field": "kubernetes.pod_owner",
        "pattern": "%{_tmp.parent_type}/%{_tmp.parent_name}",
        "ignore_missing": true
      }
    },
    { "lowercase": { "field": "_tmp.parent_type", "ignore_missing": true } },
    {
      "set": {
        "field": "kubernetes.pod.{{_tmp.parent_type}}.name",
        "value": "{{_tmp.parent_name}}",
        "if": "ctx?._tmp?.parent_type != null",
        "ignore_failure": true
      }
    },
    { "remove": { "field": ["_tmp", "kubernetes.pod_owner"], "ignore_missing": true } },
    // Scripts for normalizing annotations, namespace_labels, and labels (abbreviated for brevity; full in blog)
  ]
}
```


**Audit Logs Pipeline (`openshift-audit-2-ecs`)**:
```json
PUT _ingest/pipeline/openshift-audit-2-ecs
{
  "processors": [
    // Script to move fields under kubernetes.audit
    // Additional sets and renames for orchestrator, node, host
    // Script for normalizing audit annotations
  ]
}
```

(Full JSON in the blog for scripts.)

#### Step 3: Create Reroute Pipelines
Redirect logs to ECS data streams.

**Container Logs Reroute (`app-write-reroute-pipeline`)**:
```json
PUT _ingest/pipeline/app-write-reroute-pipeline
{
  "processors": [
    { "pipeline": { "name": "openshift-2-ecs" } },
    { "set": { "field": "event.dataset", "value": "kubernetes.container_logs" } },
    { "reroute": { "destination": "logs-kubernetes.container_logs-openshift" } }
  ]
}
```


**Audit Logs Reroute (`audit-write-reroute-pipeline`)**:
Similar structure, rerouting to `logs-kubernetes.audit_logs-openshift`.

#### Step 4: Set Default Pipelines on Indices
```json
PUT app-write { "settings": { "index.default_pipeline": "app-write-reroute-pipeline" } }
PUT audit-write { "settings": { "index.default_pipeline": "audit-write-reroute-pipeline" } }
```


#### Step 5: Create Elasticsearch User and Role
```json
PUT _security/role/logging-role
{
  "cluster": ["monitor"],
  "indices": [
    { "names": ["logs-*-*"], "privileges": ["auto_configure", "create_doc"] },
    { "names": ["app-write", "audit-write"], "privileges": ["create_doc", "read"] }
  ]
}
PUT _security/user/logging-user { "password": "password", "roles": ["logging-role"] }
```


#### Step 6: Create Secret in OpenShift
```bash
oc create secret generic es-secret -n openshift-logging \
  --from-literal=username=logging-user \
  --from-literal=password=password
# Add TLS if needed: --from-file=ca-bundle.crt=...
```


#### Step 7: Create ClusterLogging CR
```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  collection:
    type: vector
```


#### Step 8: Create ClusterLogForwarder CR
```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: external-es
      type: elasticsearch
      url: https://your-es-url:9200
      elasticsearch:
        version: 8
      secret:
        name: es-secret
  pipelines:
    - name: forward-logs
      inputRefs: [application, audit]  # Exclude infrastructure if not needed
      outputRefs: [external-es]
```


#### Step 9: Verify and Troubleshoot
- Check CR status: `oc get clusterlogforwarder instance -n openshift-logging -o yaml`.
- Authentication errors: Verify user roles.
- Pipeline issues: Ensure default pipelines are set.
- Query in Kibana: Use `logs-kubernetes.container_logs-openshift` and `logs-kubernetes.audit_logs-openshift`.

# Leveraging Elastic's Kubernetes Integration
The setup emulates Elastic Agent by normalizing logs via pipelines, enabling Kubernetes dashboards in Kibana without privileged deployments. Logs lack host/cloud metadata but support annotations for custom rerouting (e.g., `elastic.co/dataset`). Use Kibana Logs Explorer for visualization.

For ECK on OpenShift alternative, deploy via OperatorHub for native integration.
