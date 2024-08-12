# Enabling Service Graphs and L3 metrics in Grafana using Grafana Beyla, OpenTelemetry Collector, and Grafana Tempo
Service graphs for a Kubernetes cluster offer valuable insights into the system's architecture. They give a high-level summary of the system's health, showcasing error rates, latencies, and other pertinent metrics. Additionally, they provide a historical perspective on the system's topology. They are a useful tool for monitoring and diagnosing issues within your Kubernetes cluster. By visualizing the interactions between services, you can quickly identify bottlenecks, pinpoint failing components, and understand the impact of different services on overall performance. This holistic view enables more effective troubleshooting and optimization, ultimately contributing to a more resilient and efficient system.

Service graphs can be visualized in grafana using either Grafana Beyla or Cilium and Hubble. After considering both options, I decided to use Grafana Beyla. For a more detailed comparison and description of the process to set up Cilium and Hubble UI service graphs (and the possibility of visualizing the service graph in Grafana), refer to this [document](Cilium, Hubble, Grafana L7 flow and service graph.pdf).

<img width="1792" alt="Screenshot 2024-08-08 at 2 49 27 PM" src="https://github.com/user-attachments/assets/79940bcb-276e-41ca-b396-1f92b69250fd">

We will need to use Grafana Beyla, OpenTelemetry Collector, Grafana Tempo, and Prometheus to display service graphs in Grafana.
The service graph feature in grafana requires a prometheus metrics data source. However, the service graph metrics need to be derived from traces produced by Grafana Beyla, which can monitor L7 and L3 data flow. As such, we will install Beyla in the cluster and use it to automatically instrument various namespaces in the cluster. We will send metrics and traces produced by Beyla to an OpenTelemetry Collector, which will then forward the traces to a Grafana Tempo endpoint. Grafana Tempo has a pod in its distributed architecture which is known as the metrics generator. It converts the traces it receives to prometheus metrics, which we can then send to a prometheus server in the cluster. Next, in Grafana, we need to configure the Tempo service as a data source and the prometheus service as the data source that converts the traces to metrics. By simply navigating to the Exdplore tab in Grafana, and switching to the Tempo data source's service graph tab, we will be able to see a service graph like the one above.

## Steps
Create a GKE cluster.
```
gcloud container clusters create beyla-test
```
Create beyla namespace
```
kubectl create namespace beyla
```
#### 1. Deploy Beyla
Create beyla service account, cluster role and cluster role binding
```
apiVersion: v1
kind: ServiceAccount
metadata:
 name: beyla
 namespace: beyla
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: beyla
rules:
 - apiGroups: [ "apps" ]
   resources: [ "replicasets" ]
   verbs: [ "list", "watch" ]
 - apiGroups: [ "" ]
   resources: [ "pods", "services", "nodes" ]
   verbs: [ "list", "watch" ]
 - apiGroups: [""]
   resources: ["pods", "nodes"]
   verbs: ["get", "list", "watch"]
 - apiGroups: ["apps"]
   resources: ["deployments"]
   verbs: ["get", "list", "watch"]
 - apiGroups: ["monitoring.coreos.com"]
   resources: ["servicemonitors"]
   verbs: ["get", "list", "watch", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: beyla
subjects:
 - kind: ServiceAccount
   name: beyla
   namespace: beyla
roleRef:
 kind: ClusterRole
 name: beyla
 apiGroup: rbac.authorization.k8s.io
```

Now, we will deploy beyla as a daemonset. For this, we will also create a beyla configmap.
In the configmap, **layer 3 metrics** can be optionally enabled by adding the section in the YAML file indicated as the section required for the beyla_network_flow_bytes metric.

beyla-config and beyla daemonset: (Beyla 1.6.4)
```
apiVersion: v1
kind: ConfigMap
metadata:
 namespace: beyla
 name: beyla-config
data:
 beyla-config.yml: |
   # this is for beyla_network_flow_bytes metric
   network:
     enable: true
     print_flows: true
   attributes:
     kubernetes:
       enable: true
     select:
       beyla_network_flow_bytes:
         include:
           - beyla.ip
           - src.name
           - dst.port
           - k8s.src.owner.name
           - k8s.src.namespace
           - k8s.dst.owner.name
           - k8s.dst.namespace
           - k8s.cluster.name
   # end of section for beyla_network_flow_bytes
   # this will provide automatic routes report while minimizing cardinality
   routes:
     unmatched: heuristic
   #service discovery
   discovery:
     services:
       - k8s_deployment_name: "^docs$"
       - open_ports: 8080-8089
       - k8s_deployment_name: "^website$"
       - k8s_namespace: default
       - k8s_namespace: beyla
       - k8s_deployment_name: "^client-service$"
       - k8s_deployment_name: "^hello-service$"
   otel_traces_export:
     reporters_cache_len: 1024
     sampler:
       name: "always_on"
     endpoint: http://opentelemetrycollector.beyla:4317
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
 namespace: beyla
 name: beyla
spec:
 selector:
   matchLabels:
     instrumentation: beyla
 template:
   metadata:
     labels:
       instrumentation: beyla
   spec:
     serviceAccountName: beyla
     hostPID: true # mandatory!
     hostNetwork: true
     containers:
       - name: beyla
         image: grafana/beyla:1.6.4
         ports:
         - containerPort: 9092
           hostPort: 9092
           name: http-metrics
           protocol: TCP
         imagePullPolicy: IfNotPresent
         securityContext:
           capabilities:
             add:
               - BPF
               - PERFMON
               - NET_ADMIN
               - SYS_RESOURCE
           privileged: true # mandatory!
           readOnlyRootFilesystem: true
         volumeMounts:
           - mountPath: /config
             name: beyla-config
           - mountPath: /var/run/beyla
             name: var-run-beyla
         env:
           - name: BEYLA_CONFIG_PATH
             value: "/config/beyla-config.yml"
           - name: BEYLA_PROMETHEUS_PORT
             value: "9092"
           - name: BEYLA_PRINT_TRACES
             value: 'true'
           - name: OTEL_EXPORTER_OTLP_ENDPOINT
             value: http://opentelemetrycollector.beyla.svc.cluster.local:4317
     volumes:
       - name: beyla-config
         configMap:
           name: beyla-config
       - name: var-run-beyla
         emptyDir: {}
```

#### 2. Deploy OpenTelemetry Collector
The OpenTelemetry Collector will need to ingest the metrics and traces from Beyla. We will first create the service/endpoint that Beyla will send the metrics and traces to.
OpenTelemetry Collector service:
```
apiVersion: v1
kind: Service
metadata:
 name: opentelemetrycollector
 namespace: beyla
spec:
 ports:
 - name: grpc-otlp
   port: 4317
   protocol: TCP
   targetPort: 4317
 selector:
   app.kubernetes.io/name: opentelemetrycollector
 type: ClusterIP
```
Next, we will create the configmap used to deploy the collector.
Configmap:
```
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: collector-config
 namespace: beyla
data:
 collector.yaml: |
   receivers:
     otlp:
       protocols:
         grpc:
           endpoint: ${env:MY_POD_IP}:4317
         http:
           endpoint: ${env:MY_POD_IP}:4318
   processors:
     batch:
     memory_limiter:
       # 80% of maximum memory up to 2G
       limit_mib: 1500
       # 25% of limit up to 2G
       spike_limit_mib: 512
       check_interval: 5s
   extensions:
     zpages: {}
   exporters:
     # logging:
     otlp:
       endpoint: "tempo-distributor.tempo-test:4317"
       insecure: true
   service:
     extensions: [zpages]
     pipelines:
       traces/1:
         receivers: [otlp]
         processors: [memory_limiter, batch]
         exporters: [otlp]
       # metrics:
       #   receivers: [otlp]
       #   processors: [batch]
       #   exporters: [logging]
```
We also need to create a deployment for the OTEL collector.
Deployment:
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: opentelemetrycollector
 namespace: beyla
spec:
 replicas: 1
 selector:
   matchLabels:
     app.kubernetes.io/name: opentelemetrycollector
 template:
   metadata:
     labels:
       app.kubernetes.io/name: opentelemetrycollector
   spec:
     containers:
     - name: otelcol
       args:
       - --config=/conf/collector.yaml
       image: otel/opentelemetry-collector:0.18.0
       volumeMounts:
       - mountPath: /conf
         name: collector-config
     volumes:
     - configMap:
         items:
         - key: collector.yaml
           path: collector.yaml
         name: collector-config
       name: collector-config
```
#### 3. Deploy Client-service and Hello-service to generate sample traces and metrics
We will create a service called hello-service that simply returns "Hello from Hello Service" when visited. We will also create a client-service that continuously queries hello-service every 5 seconds in order to generate sample traces and metrics.
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: hello-service
 namespace: beyla
spec:
 replicas: 1
 selector:
   matchLabels:
     app: hello-service
 template:
   metadata:
     labels:
       app: hello-service
   spec:
     containers:
     - name: hello-service
       image: hashicorp/http-echo:0.2.3
       args:
       - "-text=Hello from Hello Service"
       ports:
       - containerPort: 5678
—--
apiVersion: apps/v1
kind: Deployment
metadata:
 name: client-service
 namespace: beyla
spec:
 replicas: 1
 selector:
   matchLabels:
     app: client-service
 template:
   metadata:
     labels:
       app: client-service
   spec:
     containers:
     - name: client-service
       image: curlimages/curl:7.83.1
       args:
       - "/bin/sh"
       - "-c"
       - "while true; do curl -s hello-service.beyla; sleep 5; done"
—--
apiVersion: v1
kind: Service
metadata:
 name: hello-service
 namespace: beyla
spec:
 selector:
   app: hello-service
 ports:
   - protocol: TCP
     port: 80
     targetPort: 5678
—--
apiVersion: v1
kind: Service
metadata:
 name: client-service
 namespace: beyla
spec:
 selector:
   app: client-service
 ports:
   - protocol: TCP
     port: 80
     targetPort: 80
```
#### 4. Deploy Prometheus
Create the prometheus namespace
```
kubectl create namespace prometheus
```
Create the configmap for prometheus. This configmap has been modified to enable the ```--web.enable-remote-write-receiver``` flag when deplopying prometheus, which allows prometheus to receive any metrics from the metrics generator using the prometheus remote write feature.
```
apiVersion: v1
kind: ConfigMap
metadata:
 name: prometheus-server-conf
 namespace: prometheus
data:
 prometheus.yml: |
   global:
     scrape_interval: 15s
     evaluation_interval: 15s
   scrape_configs:
     - job_name: 'beyla'
       static_configs:
         - targets: ['beyla.beyla:9092']
```
We will also create a LoadBalancer service for prometheus, so that we can access it directly for testing. You may create a ClusterIP service instead.
```
apiVersion: v1
kind: Service
metadata:
 name: prometheus-service
 namespace: prometheus
spec:
 selector:
   app: prometheus-server
 ports:
   - protocol: TCP
     port: 80
     targetPort: 9090
 type: LoadBalancer
```
Deploy prometheus as a deployment in the prometheus namespace.
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: prometheus-server
 namespace: prometheus
spec:
 replicas: 1
 selector:
   matchLabels:
     app: prometheus-server
 template:
   metadata:
     labels:
       app: prometheus-server
   spec:
     containers:
       - name: prometheus
         image: prom/prometheus
         args:
           - "--config.file=/etc/prometheus/prometheus.yml"
           - "--web.enable-remote-write-receiver"
         ports:
           - containerPort: 9090
         volumeMounts:
           - name: config-volume
             mountPath: /etc/prometheus
     volumes:
       - name: config-volume
         configMap:
           name: prometheus-server-conf
           defaultMode: 420
```
#### 5. Deploy Grafana
Deploy grafana and a LoadBalancer service for grafana.
```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: grafana
 namespace: grafana
spec:
 replicas: 1
 selector:
   matchLabels:
     app: grafana
 template:
   metadata:
     labels:
       app: grafana
   spec:
     containers:
     - name: grafana
       image: grafana/grafana
       ports:
       - containerPort: 3000
       env:
       - name: GF_SECURITY_ADMIN_PASSWORD
         value: "admin"
---
apiVersion: v1
kind: Service
metadata:
 name: grafana
 namespace: grafana
spec:
 type: LoadBalancer
 ports:
   - port: 80
     targetPort: 3000
     protocol: TCP
 selector:
   app: grafana
```
#### 6. Deploy Grafana Tempo
We will use Helm to deploy the Tempo distributed architecture.
First, create the namespace ```tempo-test```
```
kubectl create namespace tempo-test
```
Next, install tempo distributed architecture using Helm.
```
helm -n tempo-test install tempo grafana/tempo-distributed
```
The default installation does not automatically enable the Tempo metrics generator that we need. As such, we need to edit the helm release 'Values' to enable metrics generator. This process can be done easily if you have the Mirantis Lens Kubernetes IDE. 
Go to the Helm releases in Lens in the tempo-test namespace. Select the tempo release.
Edit the Values file by adding the ```defaults``` section in ```global_overrides```:
```
global_overrides:
  defaults:
    metrics_generator:
      processor:
        service_graphs: null
        span_metrics: null
      processors:
        - service-graphs
        - span-metrics
  per_tenant_override_config: /runtime-config/overrides.yaml
```
In the ```metricsGenerator``` section, we also need to set the Prometheus endpoint that the metrics will sent to. Under ```config.storage```, set the remote write URL as ```http://prometheus-service.prometheus/api/v1/write```.
Also change ```metricsGenerator.enabled``` from ```false``` to ```true```.

```metricsGenerator.config.storage```:
```
storage:
  path: /var/tempo/wal
  remote_write:
    - url: http://prometheus-service.prometheus/api/v1/write
  remote_write_add_org_id_header: true
  remote_write_flush_deadline: 1m
  wal: null
```
```metricsGenerator.enabled```:
```
enabled: true
```
Save and deploy a new release of tempo with the edited Values. Metrics Generator pods should now appear in the tempo-test namespace.

#### 7. Configure Grafana Data sources and view service graphs
Now, we need to create Tempo and Prometheus data sources in Grafana.
Open grafana using the LoadBalancer service. 

**Prometheus**

First, create the prometheus data source.
Create a data source and set the HTTP URL as ```http://prometheus-service.prometheus:80```.
Save and test the connection.

**Tempo**

Create the tempo data source using the HTTP URL ```http://tempo-query-frontend.tempo-test:3100```.
This data source requires additional settings to be configured to enable service graphs.
Tempo data source additional settings:
- Select the trace to metrics data source as 'prometheus'.
- In the 'Additional settings' section, select the service graph data source as prometheus and enable node graphs.

**Viewing Service Graphs**

Go to the Explore tab in Grafana. Click on the service graph tab in the query. To view service graphs in a dashboard, create a node graph visualization in a dashboard and set Tempo as the data source.

**Notes and Issues:**
If OTEL collector or other pods keep getting evicted/keep restarting due to memory pressure:
- increase node pool size of the gke cluster
- or remove the beyla namespace watch from the beyla configmap.

Change these settings to the following in the tempo helm chart if there is a ```ResourceExhausted``` error in the OTEL collector logs when sending traces to Tempo:
```
server:
  grpc_server_max_recv_msg_size: 123412341234123
  grpc_server_max_send_msg_size: 123412341234123
```
