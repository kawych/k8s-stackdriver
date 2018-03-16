# Custom Metrics - Stackdriver Adapter

Custom Metrics - Stackdriver Adapter implements [Custom Metrics API] and
[External Metrics API] based on Stackdriver metrics. It's purpose is to
enable pod autoscaling based on Stackdriver metrics.

## Usage guide

This guide shows how to set up Custom Metrics - Stackdriver Adapter and export
metrics to Stackdriver in a compatible way. Once this is done, you can use
them to scale your application, following [HPA walkthrough].

### Configure cluster

1. Create Kubernetes cluster or use existing one, see [cluster setup].
   Requirements:

   * Kubernetes version 1.8.1 or newer running on GKE or GCE

   * Monitoring scope `monitoring` set up on cluster nodes. **It is enabled by
     default, so you should not have to do anything**. See also [OAuth 2.0 API
     Scopes] to learn more about authentication scopes.

     You can use following commands to verify that the scopes are set correctly:
     - For GKE cluster `<my_cluster>`, use following command:
       ```
       gcloud container clusters describe <my_cluster>
       ```
       For each node pool check the section `oauthScopes` - there should be
       `https://www.googleapis.com/auth/monitoring` scope listed there.
     - For a GCE instance `<my_instance>` use following command:
       ```
       gcloud compute instances describe <my_instance>
       ```
       `https://www.googleapis.com/auth/monitoring` should be listed in the
       `scopes` section.


     To configure set scopes manually, you can use:
     - `--scopes` flag if you are using `gcloud container clusters create`
       command, see [gcloud
       documentation](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create).
     - Environment variable `NODE_SCOPES` if you are using [kube-up.sh script].
       It is enabled by default.
     - To set scopes in existing clusters you can use `gcloud beta compute
       instances set-scopes` command, see [gcloud
       documentation](https://cloud.google.com/sdk/gcloud/reference/beta/compute/instances/set-scopes).

1. Start *Custom Metrics - Stackdriver Adapter*.

```sh
kubectl create -f
https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/adapter-beta.yaml
```

### Export custom metrics to Stackdriver

To learn how to create your custom metric and write your data to Stackdriver,
follow [Stackdriver custom metrics documentation]. You can also follow
[Prometheus to Stackdriver documentation] to export metrics exposed by your pods
in Prometheus format.

The name of your metric must start with custom.googleapis.com/ prefix followed
by a simple name, as defined in [custom metric naming rules].

You may choose to use Custom Metrics - Stackdriver Adapter with old Stackdriver
resource model, or the new one.
- For old resource model, use monitored resource `gke_container`
- For new resource model, use one of Kubernetes-dedicated monitored resources:
  `k8s_pod` or `k8s_node` - corresponding to Kubernetes objects `Pod` and
  `Node`. **New resource model is not yet available**.

1. **Define your custom metric** by following [Stackdriver custom metrics documentation].
   Your metric descriptor needs to meet following requirements:
   * `metricKind = GAUGE`
   * `metricType = DOUBLE` or `INT64`
1. **Export metric from your application.** The metric has to be associated with a
   specific pod and meet folowing requirements:
   * **For old Stackdriver resource model:**
     * `resource_type = "gke_container"` (See [monitored resources documentation])
     * Following resource labels are set to correct values:
       - `pod_id` - set this to pod UID obtained via downward API. Example configuration that passes POD_ID to your application as a flag:

         ```yaml
         apiVersion: v1
         kind: Pod
         metadata:
           name: my-pod
         spec:
           containers:
           - image: <my-image>
             command:
             - my-app --pod-id=$(POD_ID)
             env:
             - name: POD_ID
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.uid
         ```

         Example flag definition in Go:
         ```go
         import "flag"

         podIdFlag := flag.String("pod-id", "", "a string")
         flag.Parse()
         podID := *podIdFlag
         ```

       - `container_name = ""`
       - `project_id`, `zone`, `cluster_name` - can be obtained by your application from
          [metadata server]. You can use Google Cloud compute metadata client to get
          these values, example in Go:

         ```go
         import (
           gce "cloud.google.com/go/compute/metadata"
           "strings"
         )

         project_id, err := gce.ProjectID()
         zone, err := gce.Zone()
         cluster_name, err := strings.TrimSpace(gce.InstanceAttributeValue("cluster-name"))
         ```

       Example code exporting a metric to Stackdriver, written in Go:

       ```go
       import (
         "context"
         "time"
         "golang.org/x/oauth2"
         "golang.org/x/oauth2/google"
         "google.golang.org/api/monitoring/v3"
       )

       // Create stackdriver client
       authClient := oauth2.NewClient(context.Background(), google.ComputeTokenSource(""))
       stackdriverService, err := v3.New(oauthClient)
       if err != nil {
         return
       }

       // Define metric time series filling in all required fields
       request := &v3.CreateTimeSeriesRequest{
         TimeSeries: []*v3.TimeSeries{
           {
             Metric: &v3.Metric{
               Type: "custom.googleapis.com/" + <your metric name>,
             },
             Resource: &v3.MonitoredResource{
               Type: "gke_container",
               Labels: map[string]string{
                 "project_id":     <your project ID>,
                 "zone":           <your cluster zone>,
                 "cluster_name":   <your cluster name>,
                 "container_name": "",
                 "pod_id":         <your pod ID>,
                 // namespace_id and instance_id labels don't matter for the custom
                 // metrics use case
                 "namespace_id":   <your namespace>,
                 "instance_id":    <your instance ID>,
               },
             },
             Points: []*v3.Point{
               &v3.Point{
                 Interval: &v3.TimeInterval{
                   EndTime: time.Now().Format(time.RFC3339),
                 },
                 Value: &v3.TypedValue{
                   Int64Value: <your metric value>,
                 },
               }
             },
           },
         },
       }
       stackdriverService.Projects.TimeSeries.Create("projects/<your project ID>", request).Do()
       ```

   * **For new Stackdriver resource model:**
     * `resource_type = "k8s_pod"` or `"k8s_node"` (See [monitored resources documentation])
     * Following resource labels are set to correct values:
       - `pod_name`, `namespace_name` with monitored resource `k8s_pod` or
         `node_name` with monitored resource `k8s_node`.
         `pod_name` and `namespace_name` can be obtained via downward API and
         passed to a pod as a flag, for example:

         ```yaml
         apiVersion: v1
         kind: Pod
         metadata:
           name: my-pod
         spec:
           containers:
           - image: <my-image>
             command:
             - my-app --pod-name=$(POD_NAME) --namespace=$(NAMESPACE)
             env:
             - name: POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
         ```

         Example flag definition in Go:
         ```go
         import "flag"

         podNameFlag := flag.String("pod-name", "", "a string")
         namespaceFlag := flag.String("namespace", "", "a string")
         flag.Parse()
         podName := *podNameFlag
         namespace = *namespaceFlag
         ```

       - `project_id`, `location`, `cluster_name` to identify a cluster. It can
         be obtained by your application from [metadata server]. You can use
         Google Cloud compute metadata client to get these values, example in Go:

         ```go
         import (
           gce "cloud.google.com/go/compute/metadata"
           "strings"
         )

         project_id, err := gce.ProjectID()
         location, err := strings.TrimSpace(gce.InstanceAttributeValue("cluster-location"))
         cluster_name, err := strings.TrimSpace(gce.InstanceAttributeValue("cluster-name"))
         ```

       Example code exporting a pod metric to Stackdriver, written in Go:

       ```go
       import (
         "context"
         "time"
         "golang.org/x/oauth2"
         "golang.org/x/oauth2/google"
         "google.golang.org/api/monitoring/v3"
       )

       // Create stackdriver client
       authClient := oauth2.NewClient(context.Background(), google.ComputeTokenSource(""))
       stackdriverService, err := v3.New(oauthClient)
       if err != nil {
         return
       }

       // Define metric time series filling in all required fields
       request := &v3.CreateTimeSeriesRequest{
         TimeSeries: []*v3.TimeSeries{
           {
             Metric: &v3.Metric{
               Type: "custom.googleapis.com/" + <your metric name>,
             },
             Resource: &v3.MonitoredResource{
               Type: "k8s_pod",
               Labels: map[string]string{
                 "project_id":     <your project ID>,
                 "location":       <your cluster location>,
                 "cluster_name":   <your cluster name>,
                 "pod_name":       <your pod name>,
                 "namespace_name": <your namespace>,
               },
             },
             Points: []*v3.Point{
               &v3.Point{
                 Interval: &v3.TimeInterval{
                   EndTime: time.Now().Format(time.RFC3339),
                 },
                 Value: &v3.TypedValue{
                   Int64Value: <your metric value>,
                 },
               }
             },
           },
         },
       }
       stackdriverService.Projects.TimeSeries.Create("projects/<your project ID>", request).Do()
       ```

### Using External Metrics API

Custom Metrics - Stackdriver Adapter exposes [External Metrics API] to allow for
autoscaling using metrics, that are not necessarily attached to any Kubernetes
object. To set up Horizontal Pod Autoscaler with External Metrics, see [this
guide](TODO). See the section below to learn how to correctly specify
Stackdriver metric for HPA.

#### Identifying Stackdriver Metrics

The metric is specified using metric name and label selector.

1. `metricName` is a name of Stackdriver Metric with `/` replaced with `|` in
   all places. This is due to a limitation in Kubernetes API Machinery - strings
   with `/` character can't be passed around i.e. for security reasons. For
   example, name `pubsub.googleapis.com|subscription|num_undelivered_messages`
   can be used to indicate metric
   `pubsub.googleapis.com/subscription/num_undelivered_messages`

1. `metricSelector` is a selector that specifies which metric should be used.
   List of permitted labels:
   - Resource type: `resource.type`, specifying monitored resource. It's
     recommended that `resource.type` is always specified with `Equals`
     operator, for example:
     ```yaml
     selector:
       matchLabels:
         resource.type: pubsub_subscription
     ```
     or
     ```yaml
     selector:
       matchExpressions:
         - {key: resource.type, operator: Equals, values: [pubsub_subscription]}
     ```
   - Resource labels - starting with `resource.label.`, for example
     `resource.label.subscription_id`. It's recommended that all resource labels
     defined for used monitored resource are specified with `Equals` operator.
     To see the list of labels, find your monitored resource in the [monitored
     resources documentation]. For example:
     ```yaml
     selector:
       matchLabels:
         resource.label.subscription_id: my_sub_id
     ```
     or
     ```yaml
     selector:
       matchExpressions:
         - {key: resource.label, operator: Equals, values: [my_pub_id]}
     ```
   - An alternative to specifying monitored resource directly via set of
     resource labels is to specify a selector for resource labels (starting with
     `resource.label.`), system labels (starting with `metadata.system_label.`)
     user labels (starting with `metadata.user_label.`) and metric labels
     (starting with `metric.label.`). This allows to:
     - Define a selector for multiple metrics, which will be then aggregated by
       HPA.
     - Identify a monitored resource by metadata different than specified as
       resource labels, using system labels or user labels instead.
     Following selector operators are supported:
     - Equals: `{key: <my-label>, operator: Equals, values: [<my-value>]}`
     - NotEquals: `{key: <my-label>, operator: NotEquals, values: [<my-value>]}`
     - Exists: `{key: <my-label>, operator: Exists, values: []}`
     - In: `{key: <my-label>, operator: In, values:
       [<my-value-1>,<my-value-2>,...]}`
     - NotIn: `{key: <my-label>, operator: NotIn, values: [<my-value-1>,<my-value-2>,...]}`
     - GreaterThan: `{key: <my-label>, operator: GreaterThan, values:
       [<my-value>]}`
     - LessThan: `{key: <my-label>, operator: LessThan, values: [<my-value>]}`
     Operator `DoesNotExist` is not supported.

Here are some real-world examples on how to correctly specify Stackdriver
metrics for HPA:

1. Use a metric for a single monitored resource:
   ```yaml
   metrics:
   - type: External
     metricName: pubsub.googleapis.com|subscription|num_undelivered_messages
     metricSelector:
       matchLabels:
       resource.type: pubsub_subscription
       resource.label.subscription_id: example
     targetAverageValue: <my-project-id>
   ```

1. Use a labeled metric for single monitored resource. Time series matching
   specified expressions will be aggregated together:

   ```yaml
   metrics:
   - type: External
     metricName: cloudtasks.googleapis.com|api|request_count
     metricSelector:
       matchLabels:
         resource.type: cloud_task_queue
         resource.label.project_id: <my-project-id>
         resource.label.queue_id: <my-queue-id>
         resource.target_type: <target-type>
       matchExpressions:
       - {key: metric.label.api_method, operator: Equals, values: [CreateTask]}
       - {key: metric.label.response_code, operator: In, values
         [unavailable,deadline_exceeded]
     targetAverageValue: 42
   ```

1. Use a metric for multiple GCE instances, selected by specifying system
   labels: 

   ```yaml
   metrics:
   - type: External
     metricName: compute.googleapis.com|instance|cpu|usage_time
     metricSelector:
       matchLabels:
         resource.type: gce_instance
         metadata.system_label.spot_instance: true
     targetAverageValue: <my-project-id>
   ```

### Examples

To test your custom metrics setup or see a reference on how to push your metrics
to Stackdriver, check out our examples:
* application that exports custom metric directly to Stackdriver filling in all
  required labels: [direct-example]
* application that exports custom metrics to Stackdriver using [Prometheus text format] 
  and [Prometheus to Stackdriver] adapter: [prometheus-to-sd-example]

[Custom Metrics API]:
https://github.com/kubernetes/metrics/tree/master/pkg/apis/custom_metrics
[Custom Metrics API]:
https://github.com/kubernetes/metrics/tree/master/pkg/apis/external_metrics
[HPA walkthrough]:
https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
[cluster setup]: https://kubernetes.io/docs/setup/
[Stackdriver custom metrics documentation]:
https://cloud.google.com/monitoring/custom-metrics/creating-metrics
[Prometheus to stackdriver documentation]:
https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/prometheus-to-sd
[custom metric naming rules]:
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md#metric-names
[direct-example]:
https://github.com/GoogleCloudPlatform/k8s-stackdriver/blob/master/custom-metrics-stackdriver-adapter/examples/direct-to-sd
[prometheus-to-sd-example]:
https://github.com/GoogleCloudPlatform/k8s-stackdriver/blob/master/custom-metrics-stackdriver-adapter/examples/prometheus-to-sd
[OAuth 2.0 API Scopes]:
https://developers.google.com/identity/protocols/googlescopes
[kube-up.sh script]:
https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh
[monitored resources documentation]:
https://cloud.google.com/monitoring/api/resources
[Prometheus to Stackdriver]:
https://github.com/GoogleCloudPlatform/k8s-stackdriver/tree/master/prometheus-to-sd
[Prometheus text format]:
https://prometheus.io/docs/instrumenting/exposition_formats
