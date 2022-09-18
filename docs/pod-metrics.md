# Pod Metrics

| Metric name| Metric type | Description | Unit (where applicable) | Labels/tags | Status | Opt-in |
| ---------- | ----------- | ----------- | ----------------------- | ----------- | ------ | ------ |
| kube_pod_annotations | Gauge | Kubernetes annotations converted to Prometheus labels | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `annotation_POD_ANNOTATION`=&lt;POD_ANNOTATION&gt; <br> `uid`=&lt;pod-uid&gt; | EXPERIMENTAL | -
| kube_pod_info | Gauge | Information about pod | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `host_ip`=&lt;host-ip&gt; <br> `pod_ip`=&lt;pod-ip&gt; <br> `node`=&lt;node-name&gt;<br> `created_by_kind`=&lt;created_by_kind&gt;<br> `created_by_name`=&lt;created_by_name&gt;<br> `uid`=&lt;pod-uid&gt;<br> `priority_class`=&lt;priority_class&gt;<br> `host_network`=&lt;host_network&gt;| STABLE | - |
| kube_pod_ips | Gauge | Pod IP addresses | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `ip`=&lt;pod-ip-address&gt; <br> `ip_family`=&lt;4 OR 6&gt; <br> `uid`=&lt;pod-uid&gt; | EXPERIMENTAL | - |
| kube_pod_start_time | Gauge | Start time in unix timestamp for a pod | seconds | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
| kube_pod_completion_time | Gauge | Completion time in unix timestamp for a pod | seconds | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
| kube_pod_owner | Gauge | Information about the Pod's owner | |`pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `owner_kind`=&lt;owner kind&gt; <br> `owner_name`=&lt;owner name&gt; <br> `owner_is_controller`=&lt;whether owner is controller&gt; <br> `uid`=&lt;pod-uid&gt;  | STABLE | - |
| kube_pod_labels | Gauge | Kubernetes labels converted to Prometheus labels | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `label_POD_LABEL`=&lt;POD_LABEL&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
| kube_pod_nodeselectors| Gauge | Describes the Pod nodeSelectors | |  `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `nodeselector_NODE_SELECTOR`=&lt;NODE_SELECTOR&gt; <br> `uid`=&lt;pod-uid&gt; | EXPERIMENTAL | Opt-in |
| kube_pod_status_phase | Gauge | The pods current phase | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `phase`=&lt;Pending\|Running\|Succeeded\|Failed\|Unknown&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
| kube_pod_status_ready | Gauge | Describes whether the pod is ready to serve requests | | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `condition`=&lt;true\|false\|unknown&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
| kube_pod_status_ready_time | Gauge | Time when pod passed readiness probes. | seconds | `pod`=&lt;pod-name&gt; <br> `namespace`=&lt;pod-namespace&gt; <br> `uid`=&lt;pod-uid&gt; | STABLE | - |
## Useful metrics queries

### How to retrieve non-standard Pod state

It is not straightforward to get the Pod states for certain cases like "Terminating" and "Unknown" since it is not stored behind a field in the `Pod.Status`.

So to mimic the [logic](https://github.com/kubernetes/kubernetes/blob/v1.17.3/pkg/printers/internalversion/printers.go#L624) used by the `kubectl` command line, you will need to compose multiple metrics.

For example:

* To get the list of pods that are in the `Unknown` state, you can run the following PromQL query: `sum(kube_pod_status_phase{phase="Unknown"}) by (namespace, pod) or (count(kube_pod_deletion_timestamp) by (namespace, pod) * sum(kube_pod_status_reason{reason="NodeLost"}) by(namespace, pod))`

* For Pods in `Terminating` state: `count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod)`

Here is an example of a Prometheus rule that can be used to alert on a Pod that has been in the `Terminated` state for more than `5m`.

```yaml
groups:
- name: Pod state
  rules:
  - alert: PodsBlockInTerminatingState
    expr: count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod) > 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: Pod {{$labels.namespace}}/{{$labels.pod}} block in Terminating state.
```
