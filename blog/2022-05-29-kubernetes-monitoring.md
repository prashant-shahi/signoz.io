---
title: Monitoring Kubernetes Guide
slug: monitoring-kubernetes
date: 2022-05-29
tags: [opentelemetry, kubernetes, infra, monitoring]
authors: prashant
description: In this blog, we take you through a hands-on guide on monitoring your Kubernetes cluster using OpenTelemetry and SigNoz.
image: /img/blog/2022/05/monitoring_kubernetes.webp
keywords:
  - OpenTelemetry
  - Kubernetes
  - Monitoring
  - Infra
---

In this blog, we take you through a hands-on guide on monitoring your
Kubernetes cluster using OpenTelemetry and SigNoz.

<!--truncate-->

![Cover Image](/img/blog/2022/05/monitoring_kubernetes.webp)

## What is Kubernetes?

Kubernetes, also known as K8s, is an open-source system for automating
deployment, scaling, and management of containerized applications. It
was released as open source in 2014 by Google, and they had been open
about <a href = "https://speakerdeck.com/jbeda/containers-at-scale" rel="noopener noreferrer nofollow" target="_blank">running everything at Google in containers</a>.

The open source project is currently hosted by the Cloud Native Computing Foundation
(<a href = "https://www.cncf.io/about" rel="noopener noreferrer nofollow" target="_blank">CNCF</a>)
and used by many companies around the world.

## Why Monitor Kubernetes Cluster?

Kubernetes makes it easy to deploy and operate applications in a
microservice architecture. However as the number of microservice
increases, it gets difficult to monitor. On top of that, traditional
monitoring tools are often inadequate for Kubernetes environment.

With the increase in adoption of container orchestration tools and
new architectures such as microservices, we can assert that
monitoring ecosystem is evolving. It now consists of state-of-the-art
concepts such as **Observability**, which is commonly divided into three
major categories — **metrics**, **traces**, and **logs** — popularly
known as the **three pillars of observability**.

## What is OpenTelemetry?

<a href = "https://opentelemetry.io/" rel="noopener noreferrer nofollow" target="_blank">OpenTelemetry</a>,
also known as OTel for short, is an open-source vendor-agnostic set of tools, APIs, and SDKs
used to instrument applications to create and manage telemetry
data(metrics, traces, and logs). It aims to make telemetry data
a built-in feature of cloud-native software applications.

The telemetry data is then sent to an observability tool for
storage and visualization.

<figure data-zoomable align='center'>
   <img src="/img/blog/common/how_opentelemetry_fits.webp" alt="How opentelemetry fits with an application"/>
   <figcaption>
      <i>
         OpenTelemetry libraries instrument application code to generate telemetry data
         that is then sent to an observability tool for storage & visualization.
      </i>
   </figcaption>
</figure>

<br></br>

OpenTelemetry is the bedrock for setting up an observability framework.
It also provides you the freedom to choose a backend analysis tool of your choice.

## OpenTelemetry and SigNoz

In this article, we will use [SigNoz](https://signoz.io/?utm_source=blog&utm_medium=monitoring-kubernetes), a full-stack
**open-source application monitoring and observability platform** that
can be used for storing and visualizing the telemetry data collected with
OpenTelemetry. It is built natively on OpenTelemetry and works on the
various data formats: OTLP, Zipkin, Jaeger, Prometheus back-ends, etc.

SigNoz provides query and visualization capabilities for the end-user and
comes with out-of-box charts for application metrics and traces.

Now let's get down to some action and see everything for yourself.

We will divide the tutorial into two parts:

1. Installing SigNoz
2. Kubernetes Infrastructure Monitoring

## Installing SigNoz

First, you need to install SigNoz so that OpenTelemetry can send the data to it.

SigNoz can be installed on Kubernetes easily using Helm:

```bash
helm repo add signoz https://charts.signoz.io

kubectl create ns platform

helm --namespace platform install my-release signoz/signoz
```

You should see similar output:

```
NAME: my-release
LAST DEPLOYED: Mon May 23 20:34:55 2022
NAMESPACE: platform
STATUS: deployed
REVISION: 1
NOTES:
1. You have just deployed SigNoz cluster:

- frontend version: '0.8.0'
- query-service version: '0.8.0'
- alertmanager version: '0.23.0-0.1'
- otel-collector version: '0.43.0-0.1'
- otel-collector-metrics version: '0.43.0-0.1'
```

:::info
For detailed instructions to set up SigNoz cluster in kubernetes, refer to
[Install Kubernetes Section](https://signoz.io/docs/install/kubernetes/?utm_source=blog&utm_medium=monitoring-kubernetes).
:::

You can visit our documentation for instructions on how to install SigNoz using Docker and Helm Charts.

[![Deployment Docs](/img/blog/common/deploy_docker_documentation.webp)](https://signoz.io/docs/install/?utm_source=blog&utm_medium=monitoring-kubernetes)

To port forward SigNoz UI on your local machine, run the following:

```bash
kubectl port-forward -n platform service/my-release-signoz-frontend 3301
```

When you are done installing SigNoz, you can access the UI at
[http://localhost:3301](http://localhost:3301/application).

:::info
You can alternatively set SigNoz Frontend service type as `LoadBalancer`/`NodePort`
or use `Ingress` for custom domain.
:::

<br></br>

Now that you have SigNoz up and running, let's set up our OtelCollectors to collector
kubernetes cluster metrics.

## Kubernetes Infrastructure Monitoring

We will use the following receivers of OpenTelemetry collector:
- `kubeletstats`: Kubelet Stats Receiver pulls pod metrics from the API server on a kubelet
- `hostmetrics`: Host Metrics receiver generates metrics about the host system scraped

### Steps to export k8s metrics to SigNoz

1. **Clone Otel collector repo**

   ```bash
   git clone https://github.com/SigNoz/otel-collector-k8s.git && cd otel-collector-k8s
   ```

2. **Set up the address to SigNoz in your OTel collectors**

   You need to set up the address to SigNoz in your OTel collector which is
   collecting the k8s metrics.

   a. If you are running SigNoz in an independent Kubernetes cluster or VM, you need to change
   the placeholder IPs in the following files with the IP of machine
   where you are hosting signoz.
   
   -  [agent/infra-metrics.yaml](https://github.com/SigNoz/otel-collector-k8s/blob/main/agent/infra-metrics.yaml#L47) 
   -  [deployment/all-in-one.yaml](https://github.com/SigNoz/otel-collector-k8s/blob/main/deployment/all-in-one.yaml#L19)  

   You need to update the below section.
   
   ```yaml
   exporters:
      otlp:
        endpoint: "<SigNoz-Otel-Collector-Address>:4317"
        tls:
          insecure: true
   ```

   b. If you are running SigNoz in the same Kubernetes cluster where
   your applications are, you have to replace the above endpoint in
   [agent/infra-metrics.yaml](https://github.com/SigNoz/otel-collector-k8s/blob/main/agent/infra-metrics.yaml#L47)
   and [deployment/all-in-one.yaml](https://github.com/SigNoz/otel-collector-k8s/blob/main/deployment/all-in-one.yaml#L19) by

   ```
   my-release-signoz-otel-collector.platform.svc.cluster.local:4317
   ```

   :::info
   - `my-release` is the Helm release name
   - `platform` is the namespace where SigNoz is deployed
   - In case of SigNoz installed in different kubernetes cluster/machine,
   update it to appropriate address.
   :::

3. **Install OTel collectors and enable specific receivers to send metrics to SigNoz**
   
   To access metrics from kubeletstats receivers you have to:

   ```bash
   kubectl create ns signoz-infra-metrics
   kubectl -n signoz-infra-metrics apply -Rf agent
   kubectl -n signoz-infra-metrics apply -Rf deployment
   ```

   Output:
   ```
   namespace/signoz-infra-metrics created
   daemonset.apps/otel-collector-agent created
   configmap/otel-collector-agent-conf created
   serviceaccount/sa-otel-agent created
   clusterrole.rbac.authorization.k8s.io/sa-otel-agent-role created
   clusterrolebinding.rbac.authorization.k8s.io/aoc-agent-role-binding created
   configmap/otelcontribcol created
   serviceaccount/otelcontribcol created
   clusterrole.rbac.authorization.k8s.io/otelcontribcol created
   clusterrolebinding.rbac.authorization.k8s.io/otelcontribcol created
   deployment.apps/otelcontribcol created
   ```

   To check pod status:

   ```bash
   kubectl -n signoz-infra-metrics get pods
   ```

   Output:
   ```
   NAME                             READY   STATUS    RESTARTS   AGE
   otel-collector-agent-kkchn       1/1     Running   0          2m
   otelcontribcol-6d45c844c-tk2k8   1/1     Running   0          2m
   ```

   To check logs of the OTel collector agent:

   ```bash
   export POD_NAME=$(kubectl -n signoz-infra-metrics get pods -l "component=otel-collector-agent" -o jsonpath="{.items[0].metadata.name}")

   kubectl -n signoz-infra-metrics logs $POD_NAME
   ```

   Output should look like this:

   ```
   ...
   2022-05-27T19:37:14.158Z	info	service/telemetry.go:95	Setting up own telemetry...
   2022-05-27T19:37:14.159Z	info	service/telemetry.go:115	Serving Prometheus metrics	{"address": ":8888", "level": "basic", "service.instance.id": "50674c90-240c-4e38-8c18-d2c2b8df1532", "service.version": "latest"}
   2022-05-27T19:37:14.159Z	info	service/collector.go:229	Starting otelcol-contrib...	{"Version": "0.43.0", "NumCPU": 8}
   2022-05-27T19:37:14.159Z	info	service/collector.go:124	Everything is ready. Begin running and processing data.
   ```

   :::info
   - In case of any errors in the above logs, you should not see except for the case of
   SigNoz being unavailable or inaccessible.
   :::

4. **Plot Metrics in SigNoz UI**

   If the previous step was a success, you should be able to plot graphs from the
   [list of kubelet metrics](https://signoz.io/docs/tutorial/kubernetes-infra-metrics/#list-of-metrics-from-kubernetes-receiver/?utm_source=blog&utm_medium=monitoring-kubernetes), follow
   [these instructions](https://signoz.io/docs/userguide/dashboards/?utm_source=blog&utm_medium=monitoring-kubernetes)
   to create dashboards and widgets.

### Monitor Kubelet Metrics

You can easily get started with
[this dashboard](https://github.com/SigNoz/benchmark/raw/main/dashboards/k8s-infra-metrics/cpu-memory-metrics.json).
with CPU and Memory metrics of K8s cluster containers.

To import the dashboard, go to SigNoz UI > Dashboards > New Dashboard > Import JSON.
Now, paste the above dashboard JSON and you are set!

![CPU and Memory Metrics Dashboard](/img/blog/2022/05/monitoring_kubernetes_hostmetrics.png)

You can include more widgets using other metrics to the dashboard as per your requirements.

### Monitor Node Metrics

Node metrics are very important as we have nodes underneath the abstraction
of kubernetes container orchestration.

Similar to previous section, we will be importing Hostmetrics dashboards.
However, there can be many nodes in a K8s cluster. Hence, we will be
creating multiple dashboards for each node. In future, we would add
support for label widgets which should make it possible using single
dashboard.

Let's run the following commands to automatically generate hostmetrics
dashboard JSON files for each nodes:

```bash
for node in $(kubectl get nodes -o name | xargs -n1 echo | sed -e "s/^node\///");
do
  curl -sL https://github.com/SigNoz/benchmark/raw/main/dashboards/hostmetrics/hostmetrics-import.sh \
    | HOSTNAME="$node" DASHBOARD_TITLE="HostMetrics Dashboard for $node" bash
done
```

After importing the generated dashboard JSON, you should be able to see something like this:

![Node Metics - Hostmetrics Dashboard](/img/blog/2022/05/monitoring_kubernetes_metrics.png)

---

## Conclusion

Using OpenTelemetry and SigNoz, you can set up a robust monitoring framework for
your Kubernetes cluster.

OpenTelemetry is the future for setting up observability for cloud-native apps.
It is backed by a huge community and covers a wide variety of technology and
frameworks. Using OpenTelemetry, engineering teams can easily monitor their
infrastructure and application, instrument polyglot and distributed applications
with peace of mind.

You can then use SigNoz to store and visualize your telemetry data. SigNoz is
an open-source observability tool that comes with a SaaS-like experience.
You can try out SigNoz by visiting its GitHub repo 👇 

[![SigNoz GitHub repo](/img/blog/common/signoz_github.webp)](https://github.com/SigNoz/signoz)

If you have any questions or need any help in setting things up, join our slack community and ping us in `#support` channel.

[![SigNoz Slack community](/img/blog/common/join_slack_cta.png)](https://signoz.io/slack)

---

## Further Reading

- [OpenMetrics vs OpenTelemetry](https://signoz.io/blog/openmetrics-vs-opentelemetry/?utm_source=blog&utm_medium=monitoring-kubernetes)
- [SigNoz - an open-source alternative to DataDog](https://signoz.io/blog/open-source-datadog-alternative/?utm_source=blog&utm_medium=monitoring-kubernetes)
