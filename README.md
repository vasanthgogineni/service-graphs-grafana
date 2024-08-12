# Enabling Service Graphs and L3 metrics in Grafana using Grafana Beyla, OpenTelemetry Collector, and Grafana Tempo
Service graphs for a kubernetes cluster can

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
# path/to/beylaserviceacc.yaml
```

Now, we will deploy beyla as a daemonset. For this, we will also create a beyla configmap.
In the configmap, **layer 3 metrics** can be optionally enabled by adding the section in the YAML file indicated as the section required for the beyla_network_flow_bytes metric.
beyla-config and beyla daemonset: (Beyla 1.6.4)
```
# path/to/beylaconfiganddaemonset.yaml
```

#### 2. Deploy OpenTelemetry Collector
The OpenTelemetry Collector will need to ingest the metrics and traces from Beyla. We will first create the service/endpoint that Beyla will send the metrics and traces to.
OpenTelemetry Collector service:
```
# path to OTELCOL svc
```
Next, we will create the configmap used to deploy the collector.
Configmap:
```
#path to configmap
```
We also need to create a deployment for the OTEL collector.
Deployment:
```
# path to deployment
```
#### 3. Deploy Client-service and Hello-service to generate sample traces and metrics
We will create a service called hello-service that simply returns "Hello from Hello Service" when visited. We will also create a client-service that continuously queries hello-service every 5 seconds in order to generate sample traces and metrics.
```
path to client and hello service
```
#### 4. Deploy Prometheus
Create the prometheus namespace
```
kubectl create namespace prometheus
```
Create the configmap for prometheus. This configmap has been modified to enable the ```--web.enable-remote-write-receiver``` flag when deplopying prometheus, which allows prometheus to receive any metrics from the metrics generator using the prometheus remote write feature.
```
path to prometheus configmap
```
We will also create a LoadBalancer service for prometheus, so that we can access it directly for testing. You may create a ClusterIP service instead.
```
path to prom service
```
Deploy prometheus as a deployment in the prometheus namespace.
```
path to prom deployment
```
#### 5. Deploy Grafana
Deploy grafana and a LoadBalancer service for grafana.
```
#path to grafana and service
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
