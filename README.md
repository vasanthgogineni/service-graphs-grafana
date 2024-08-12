# Enabling Service Graphs in Grafana using Grafana Beyla, OpenTelemetry Collector, and Grafana Tempo
Service graphs for a kubernetes cluster can

<img width="600" alt="Screenshot 2024-08-08 at 2 59 53 PM" src="https://github.com/user-attachments/assets/79e2b1b6-dfec-4742-a94c-bc9204719e68">

We will need to use Grafana Beyla, OpenTelemetry Collector, Grafana Tempo, and Prometheus to display service graphs in Grafana.
The service graph feature in grafana requires a prometheus metrics data source. However, the service graph metrics need to be derived from traces produced by Grafana Beyla, which can monitor L7 and L3 data flow. As such, we will install Beyla in the cluster and use it to automatically instrument various namespaces in the cluster. We will send metrics and traces produced by Beyla to an OpenTelemetry Collector, which will then forward the traces to a Grafana Tempo endpoint. Grafana Tempo has a pod in its distributed architecture which is known as the metrics generator. It converts the traces it receives to prometheus metrics, which we can then send to a prometheus server in the cluster. Next, in Grafana, we need to configure the Tempo service as a data source and the prometheus service as the data source that converts the traces to metrics. By simply navigating to the Exdplore tab in Grafana, and switching to the Tempo data source's service graph tab, we will be able to see a service graph like the one above.

