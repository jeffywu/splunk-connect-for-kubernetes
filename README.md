# Splunk Connect for Kubernetes #

Splunk Connect for Kubernetes (the connector) is a set of Kubernetes objects which once deployed in to a Kubernetes cluster, it collects data from the cluster, and sends the data to a Splunk indexer or a Splunk indexer cluster. The connector utilizes Fluentd and Heapster to collect data. Data it collects include: logs, objects, and metrics.

The connector is provided as a Helm chart, so that it can be installed easily using the Helm command line tool. Please see [the README file](./helm-chart/README.md) for the details.

For people who do not use Helm, a set of manifests manifests
=======
# What does Splunk Connect for Kubernetes do?

Splunk Connect for Kubernetes provides a way to import and search your Kubernetes logging, object, and metrics data in Splunk. Splunk is a proud contributor to Cloud Native Computing Foundation (CNCF) and Splunk Connect for Kubernetes utilizes and supports multiple CNCF components in the development of these tools to get data into Splunk.


## Prerequisites

* Splunk Enterprise 7.0 or later
* An HEC token. See the following topics for more information:
  * http://docs.splunk.com/Documentation/Splunk/7.0.3/Data/UsetheHTTPEventCollector
  * http://docs.splunk.com/Documentation/Splunk/7.0.3/Data/ScaleHTTPEventCollector
* You should be familiar with your Kubernetes configuration and know where your log info is collected in Kubernetes.
* You must have administrator access to your Kubernetes cluster. 
* To install using Helm (recommended), make sure you are running Helm in your Kubernetes configuration. See https://github.com/kubernetes/heapster
* Have a minimum of two Splunk indexes ready to collect the log data, one for both logs and Kubernetes objects, and one for metrics. You can also create separate indexes for logs and objects, in which case you will need three Splunk indexes.

## Before you begin
Splunk Connect for Kubernetes supports installation using Helm. Ensure that you thoroughly read the Prerequisites and Installation and Deployment documentation before you start your deployment of Splunk Connect for Kubernetes. 

Make sure you do the following before you install:

1. Create a minimum of two Splunk indexes: 
* one events index, which will handle logs and objects (you may also create two separate indexes for logs and objects).
* one metrics index. 
If you do not configure these indexes, Kubernetes Connect for Splunk uses the defaults created in your HEC token.

2. Create a HEC token if you do not already have one. If you are installing the connector on Splunk Cloud, file a ticket with Splunk Customer Service and they will deploy the indexes for your environment and generate your HEC token.

## Deploy with Helm

Helm, maintained by the CNCF, allows the Kubernetes administrator to install, upgrade, and manage the applications running in their Kubernetes clusters.  For more information on how to use and configure Helm Charts, please the the Helm [site](https://helm.sh/) and [repository](https://github.com/kubernetes/helm) for tutorials and product documentation. Helm is the   only method that Splunk supports for installing Splunk Connect for Kubernetes.

To install and configure defaults with Helm:

```$Helm install – name my-release - f my_values,yamlstable/splunk-connector/kubernetes-objects```

To learn more about using and modifying charts, see: https://github.com/splunk/splunk-connect-for-kubernetes/tree/master/helm-chart and https://docs.helm.sh/using_helm/#using-helm.

## Deploy using YAML

You can use YAML to `grep` the chart and manifest files and add them to your Kubernetes cluster. Please note that installation and debugging for Splunk Connect for Kubernetes through YAML is community-supported only.

When you use YAML to install Splunk Connect for Kubernetes, the installation does not create the default configuration that is created when you install using Helm. To deploy the connector using YAML, you must know how to configure your Kubernetes variables to work with the connector. If you are not familiar with this process, we recommend that you use the Helm installation method. 

To create YAML files in your Kubernetes cluster:

1. `grep` the Charts and Manifest files from https://github.com/splunk/splunk-connect-for-kubernetes

2. Apply the Charts file:

    ```kubectl apply -f charts```

3. Apply the Manifest manifest file:

    ```kubectl apply -f manifests```

Note that you may need to verify that your Kubernetes logs are recognized by the Splunk Connect for Kubernetes. See the following resources for YAML configuration properties:

* https://github.com/splunk/splunk-connect-for-kubernetes/blob/master/helm-chart/charts/splunk-kubernetes-logging/values.yaml for information about varaible configuration using YAML.
* charts/splunk-kubernetes-logging/values.yaml for configurable parameters for splunk-kubernetes-logging.
* charts/splunk-kubernetes-objects/values.yaml for configurable parameters for splunk-kubernetes-objects.
* charts/splunk-kubernetes-metrics/values.yaml for configurable parameters for splunk-kubernetes-metrics.

## Confiuration variables

For a full list of configuration variables see the following file:

https://github.com/splunk/splunk-connect-for-kubernetes/blob/master/helm-chart/charts/splunk-kubernetes-logging/values.yaml

# Architecture

Splunk Connect for Kubernetes deploys a daemonset on each node. And in the daemonset, a Fluentd container runs and does the collecting job. Splunk Connector for Kubernetes collects three types of data:

* logs: Splunk Connectr for Kubernetes collects two types of logs:
  * logs from Kubernetes system components (https://kubernetes.io/docs/concepts/overview/components/)
  * applications (container) logs
* [objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
* metrics

To collect the data, Splunk leverages:

* [Fluentd](https://www.fluentd.org/)
* [JQ plugin](https://rubygems.org/gems/fluent-plugin-jq) for transforming data
* [Splunk HEC output plug-in](https://github.com/splunk/fluent-plugin-splunk-hec): The [HTTP Event Collector](http://dev.splunk.com/view/event-collector/SP-CAAAE6M) collects all data sent to Splunk for indexing.
* For Splunk Connect for Kubernetes, Splunk uses the [node logging agent](https://kubernetes.io/docs/concepts/cluster-administration/logging/#using-a-node-logging-agent) method. See the [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) for an overview of the types of Kubernetes logs from which you may wish to collect data as well as information on how to set up those logs. 

## Logs

Splunk Connect for Kubernetes uses the Kubernetes [node logging agent](https://kubernetes.io/docs/concepts/cluster-administration/logging/#using-a-node-logging-agent) to collect logs. Splunk deploys a daemonset on each of these nodes. Each daemonset holds a Fluentd container to collect the data. The following plugins are enabled in that Fluentd container:

* [in_systemd](https://rubygems.org/gems/fluent-plugin-systemd) reads logs from systemd journal if systemd is available on the host.
* [in_tail](https://docs.fluentd.org/v1.0/articles/in_tail) reads logs from file system.
* [filter_jq_transformer](https://rubygems.org/gems/fluent-plugin-jq) transforms the raw events to a Splunk-friendly format and generates source and sourcetypes. 
* [out_splunk_hec](https://github.com/splunk/fluent-plugin-splunk-hec) sends the translated logs to Splunk indexes through the HTTP Event Collector input (HEC).

## Kubernetes Objects

Splunk Connect for Kubernetes collects Kubernetes objects that can help users access cluster status. Splunk deploys code in the Kubernetes cluster that collects the object data. That deployment contains one pod that runs Fluentd which contains the following plugins to help push data to Splunk:

* [in_kubernetes_objects](https://github.com/splunk/fluent-plugin-kubernetes-objects) collects object data by calling the Kubernetes API (by https://github.com/abonas/kubeclient). in-kubernetes-objects supports two modes:
  * watch mode: the Kubernetes API sends new changes to the plugin. In this mode, only the changed data is collected.
  * pull mode: the plugin queries the Kubernetes API periodically. In this mode, all data is collected.
* [filter_jq_transformer](https://rubygems.org/gems/fluent-plugin-jq) transforms the raw data into a Splunk-friendly format and generates sources and sourcetypes. 
* [out_splunk_hec](https://github.com/splunk/fluent-plugin-splunk-hec) sends the data to Splunk via HTTP Event Collector input (HEC).

## Metrics

Splunk Connect for Kubernetes deploys code on the Kubernetes cluster. This deployment has exactly one pod, which runs two containers:

* [Heapster](https://github.com/kubernetes/heapster) collects metrics and sends them to the Fluentd sidecar via UDP in `statsd` format.
* Fluentd, which receives metrics from Heapster using [in_udp](https://docs.fluentd.org/v1.0/articles/in_udp) and transforms the metrics using filter_jq_transformer. filter_jq_transformer formats the data for Splunk ingestion: It makea sure the metrics have proper metric_name, dimensions, etc., and then sends the metrics to Splunk using out_splunk_hec.

Make sure your Splunk configuration has a metrics index that is able to receive the data. See [Get started with metrics](http://docs.splunk.com/Documentation/Splunk/7.1.0/Metrics/GetStarted) in the Splunk Enterprise documentaiton.

If you want to learn more about how metrics are monitored in a Kubernetes cluster, see Tools for [Monitoring Compute, Storage, and Network Resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/).

# Performance

Some parameters used with Splunk Connect for Kubernetes can have an impact on overall performance of log ingestion, objects, or metrics. In general, the more filters that are added to one of the streams, the greater the preformance impact. 

By default, HEC can support up to 10K events per second with HTTP Keep-Alive disabled on clients. There are other use cases where HTTP Keep-Alive can be enabled for higher event rate, but cannot be enabled when connected to Splunk Connect for Kubernetes.

Splunk Connect for Kubernetes can support an indexing rate of over 12 MB/s indexing with over 10K events/sec and a 1 KiB message size, assuming no filters and a consistent stream of events. This means that Splunk Connect for Kubernetes can exceed the default throughput of HEC. To best address capacity needs, Splunk recommends that you monitor the HEC throughput and back pressure on Splunk Connect for Kubernetes deployments and be prepared to add additional nodes as needed.

# Processing Multi-Line Logs

One possible filter option is to enable the processing of multi-line events. This feature is currently experimental and considered to be community supported. 