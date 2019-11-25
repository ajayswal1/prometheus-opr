Monitoring

    Prometheus (Suggested)
        Prometheus Operator for Monitoring and Alerting
        Set Up Cluster-Wide Monitoring
        Monitoring Your Own Applications
            Prometheus
            Alertmanager
            Grafana
        Custom Metrics
        Ingress Controller Monitoring
    IBM Cloud Monitoring Service
        IBM Cloud Monitoring Service as Solution
            Sending Custom Metrics
            Retrieving Metrics
        Container Service Monitoring
            Configuring Metrics in Grafana
            Tutorials

Prometheus (Suggested)

Prometheus can be used to gather metrics, create graphs, and do alerting. While you do have to manage the deployment (until it is offered as a service), Prometheus allows for an extensive set of metrics, graphs, and alerts -- vastly richer and more customizable than what is natively available from the IBM Cloud Monitoring Service. It has thousands of node-level OS metrics, Kubernetes metrics, container metrics, and service metrics.

    Most of the metrics can be graphed (including through Grafana) and directly in its expression browser.
    Alertmanager can improve the signal-to-noise ratio for alerts by batching, filtering, and setting up rules for how an alert should be responded to. One common way to respond to an alert is through a Webhook.
    Applications can export custom metrics to Prometheus (such as the JMX Exporter and several others).
    Prometheus is also mentioned in the ArmadaKubProfiles for monitoring.

Prometheus Operator for Monitoring and Alerting

To set up monitoring on your Kubernetes cluster, you will first set up the Prometheus Operator and its included cluster-wide monitoring; then, you will optionally set up an individualized Prometheus-Alertmanager-Grafana stack for monitoring custom metrics from your own applications. The first step is achieved using the Prometheus Operator chart, which will install the Prometheus Operator along with some tools for basic cluster-wide monitoring. This includes:

    the Prometheus Operator itself
    a Prometheus instance
    an Alertmanager instance
    node exporters
    Prometheus rules for cluster monitoring
    Grafana pre-populated with cluster monitoring dashboards

The next section describes configuring and installing the Operator and cluster-wide monitoring; the following one explains how to use the Operator to set up monitoring for your own applications.
Set Up Cluster-Wide Monitoring

In many cases, the default configuration is enough to get started. However, a couple simple tweaks are helpful. Create a prometheus-operator-overrides.yaml file and add the following blocks to it:

    Disable monitoring of the Kubernetes scheduler and controller manager. If you don't include these lines, you will end up with an alert that is always alerting on IKS and other platforms where the control plane components are not managed by Kubernetes.

    kubeScheduler:
        enabled: false
    kubeControllerManager:
      enabled: false

    By default, the cluster-wide Prometheus instance is configured to only watch service monitors created by the Prometheus Operator chart. However, you will probably want to set up additional applications to be monitored by the cluster Prometheus (like the IBM Cloud Ingress Exporter). Let's configure a custom selector, so that we can add a label to any service monitor we create and it will be automatically picked up by the cluster Prometheus. You can choose any label(s) you want; this example matches service monitors labeled prometheus=kube-prometheus.

    commonLabels:
      prometheus: kube-prometheus
    prometheus: # You'll need to merge this block with the `prometheus` block in the next step.
      prometheusSpec:
        serviceMonitorSelector:
          matchLabels:
            prometheus: kube-prometheus

    The Prometheus Operator comes with a service monitor designed to monitor the kubelets. However, IKS does not expose kubelet metrics in the way this service monitor expects, so it is necessary to disable the default one and replace it with our own.

    kubelet:
      enabled: false
    prometheus:
      additionalServiceMonitors:
        - name: kubelet
          endpoints:
            - port: http-metrics
              honorLabels: true
              bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
              relabelings:
                - targetLabel: __address__
                  replacement: kubernetes.default.svc.cluster.local:443
                - targetLabel: __scheme__
                  replacement: https
                - sourceLabels:
                    - __meta_kubernetes_endpoint_address_target_name
                  targetLabel: __metrics_path__
                  regex: (.+)
                  replacement: /api/v1/nodes/${1}/proxy/metrics
              tlsConfig:
                caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            - port: http-metrics
              honorLabels: true
              bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
              relabelings:
                - targetLabel: __address__
                  replacement: kubernetes.default.svc.cluster.local:443
                - targetLabel: __scheme__
                  replacement: https
                - sourceLabels:
                    - __meta_kubernetes_endpoint_address_target_name
                  targetLabel: __metrics_path__
                  regex: (.+)
                  replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
              tlsConfig:
                caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          jobLabel: k8s-app
          namespaceSelector:
            matchNames:
              - kube-system
          selector:
            matchLabels:
              k8s-app: kubelet

    In order to make all metrics from local Prometheus instances available to Kubernetes's custom metrics API, set the cluster-wide instance to aggregate the metrics from other instances by adding another service monitor. You can add this directly below the snippet from the previous step.

        - name: user-prometheus
          namespaceSelector:
            any: true
          selector:
            matchLabels:
              app: prometheus
          endpoints:
            - port: http
              path: /federate
              honorLabels: true
              params:
                match[]:
                  - '{__name__=~".+"}'

    Optionally, configure Alertmanager. In order for Alertmanager to be useful, we need to set up an alerting channel. Alerts fired by Prometheus will by routed by Alertmanager according to this configuration. If you don't want to set up alerting, you can leave this block out. The configuration here applies only to alerts for cluster-wide metrics; per-application alerting settings will be configured below using an identical block.

    The example below is for alerting to a Slack channel, but there is built-in support for a variety of other platforms that can be configured similarly. To use the Slack integration, you will need to first set up an incoming webhook in Slack, and include its URL here.

    alertmanager:
      config:
        global:
          slack_api_url: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
        route:
          group_wait: 30s
          group_interval: 5m
          repeat_interval: 12h
          receiver: default
          routes:
            - match:
                alertname: DeadMansSwitch
              repeat_interval: 5m
              receiver: deadmansswitch
        receivers:
          - name: default
            slack_configs:
              - channel: my-team-alerts
                send_resolved: true
          - name: deadmansswitch

    You can optionally set the administrator password for Grafana. If you don't, then the password prom-operator will be used by default. If you configure a password here, use git-secret or another Git encryption plugin to avoid committing unencrypted secrets.

    grafana:
      adminPassword: p@ssw0rd

Once you've created the desired configuration file, install the Helm chart:

helm install --namespace monitoring -n prometheus stable/prometheus-operator -f prometheus-operator-overrides.yaml

You should now be able to see pods for the node exporters, Prometheus, Alertmanager, and Grafana:

kubectl -n monitoring get pod -l release=prometheus

You can access them by port-forwarding the following ports of the relevant pods:

    Prometheus: 9090
    Alertmanager: 9093
    Grafana: 3000

These pods are only handling cluster-wide metrics, such as node status, control plane status, and pod resource usage. To add application-specific metrics, you will create your own instances of these pods using the Prometheus Operator you just installed.
Monitoring Your Own Applications

To add monitoring to your own Helm-deployed application, the easiest thing to do is add the Prometheus, Alertmanager, and Grafana Helm charts as dependencies of your application chart. This will create a Prometheus instance that monitors only your own applications, an Alertmanager instance that deals only with alerts coming from that Prometheus instance, and a Grafana instance that pulls from your Prometheus instance and optionally from the cluster-wide Prometheus as well. This allows the monitoring and alerting for your application to be set up completely independently from the cluster-wide monitoring stack.

Add the CoreOS Helm repository to your configuration:

helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/

Add the following dependencies in your requirements.yaml, and then run helm dep up to download them:

dependencies:
  - name: prometheus
    version: 0.0.51
    repository: "@coreos"
  - name: alertmanager
    version: 0.1.7
    repository: "@coreos"
  - name: grafana
    version: 1.17.6
    repository: "@stable"

Next, Prometheus, Alertmanager, and Grafana must be configured. As usual, each subchart's configuration must go in values.yaml in a YAML block keyed by the name of the subchart. Before going into the details, here's the complete configuration:

prometheus:
  replicaCount: 2
  retention: 24h
  storageSpec:
    volumeClaimTemplate:
      metadata:
        annotations:
          volume.beta.kubernetes.io/storage-class: ibmc-file-silver
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 40Gi
  serviceMonitors:
    - name: cassandra
      selector:
        matchLabels:
          app: cassandra
          release: mychart
          role: prometheus-monitoring
      endpoints:
        - port: metrics
  rules:
    value:
      - name: default
        rules:
          - alert: InstanceDown
            expr: up == 0
            for: 5m
            annotations:
              summary: "Pod {{ $labels.pod }} down"
              description: "Pod {{ $labels.pod }} has been down for more than 5 minutes."
          - alert: ScrapeSlow
            expr: avg_over_time(scrape_duration_seconds[5m]) > 2
            for: 5m
            annotations:
              summary: "Pod {{ $labels.pod }} slow to respond"
              description: "Pod {{ $labels.pod }} has taken more than 2 seconds on average to provide metrics over the last 5 minutes."

alertmanager:
  config:
    global:
      slack_api_url: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
    route:
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: default
      routes:
        - match:
            alertname: DeadMansSwitch
          repeat_interval: 5m
          receiver: deadmansswitch
    receivers:
      - name: default
        slack_configs:
          - channel: my-team-alerts
            send_resolved: true
      - name: deadmansswitch

grafana:
  dashboardsConfigMaps:
    default: myrelease-grafana-custom
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: "default"
          orgId: 1
          folder: ""
          type: file
          disableDeletion: false
          updateIntervalSeconds: 10
          editable: true
          options:
            path: /var/lib/grafana/dashboards/default
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: "http://myrelease-prometheus:9090"
          access: proxy
          basicAuth: false
          isDefault: true
        - name: Cluster Prometheus
          type: prometheus
          url: "http://prometheus-prometheus-oper-prometheus.monitoring:9090"
          access: proxy
          basicAuth: false
  adminPassword: p@ssw0rd

For full lists of the options available for each subchart, check the default values.yaml files linked in each section below.

Prometheus

Default values.yaml

You may want to set the number of replicas and metric retention period:

replicaCount: 2
retention: 24h

To enable persistent storage, specify a persistent volume claim template. Until Thanos becomes better-integrated with the Prometheus Operator, this is probably desirable.

storageSpec:
  volumeClaimTemplate:
    metadata:
      annotations:
        volume.beta.kubernetes.io/storage-class: ibmc-file-silver
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 40Gi

IBM Cloud storage provisioners by default create new PVCs with write access for the root user only. This causes permissions problems for Prometheus, which runs as nobody (or prometheus in older versions of the Operator). There is currently no elegant solution to this, so my recommendation is to create a job as part of your chart that will run as a Helm hook as soon as the install happens that adjusts the permissions on the volume. For example:

apiVersion: batch/v1
kind: Job
metadata:
  name: pvc-permissions
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pvc-permissions
          image: busybox
          command: ["chown", "-R", "65534:65534", "/mnt/prometheus"]
          volumeMounts:
            - name: prometheus
              mountPath: /mnt/prometheus
      volumes:
        - name: prometheus
          persistentVolumeClaim:
            claimName: my-prometheus-pvc

65534 is the UID of the nobody user in the Prometheus pods, so this command will give it access to the PVC. If you're not concerned about security and want a more robust solution, you can use chmod 777 instead.

Alternately, you can manually create a PVC in advance so that it won't be recreated every time you redeploy, and manually set the permissions once.

The sources from which to pull metrics are specified in the serviceMonitors section. Each item in the list specifies a service and port from which to pull metrics.

The serviceMonitors section replaces the old way of using pod annotations to determine how to scrape pods.

serviceMonitors:
  - name: cassandra
    selector:
      matchLabels:
        app: cassandra
        release: mychart
        role: prometheus-monitoring
    endpoints:
      - port: metrics

This will pull metrics from the metrics endpoint of the service matching the given labels, if one exists.

Alerting rules are configured in the rules section. Here is a simple example. expr can be an arbitrary PromQL expression. The alert will fire if this expression evaluates to nonzero for longer than the time specified in for.

rules:
  value:
    - name: default
      rules:
        - alert: InstanceDown
          expr: up == 0
          for: 5m
          annotations:
            summary: "Pod {{ $labels.pod }} down"
            description: "Pod {{ $labels.pod }} has been down for more than 5 minutes."
        - alert: ScrapeSlow
          expr: avg_over_time(scrape_duration_seconds[5m]) > 2
          for: 5m
          annotations:
            summary: "Pod {{ $labels.pod }} slow to respond"
            description: "Pod {{ $labels.pod }} has taken more than 2 seconds on average to provide metrics over the last 5 minutes."

Alertmanager

Default values.yaml

The configuration for Alertmanager is identical to that described for cluster-wide monitoring above. If you want to send alerts to Slack or another platform, you will have to configure it for this instance as well.
Grafana

Default values.yaml

Begin by creating a Kubernetes config map containing the dashboards you wish to deploy. Obtain the JSON exports of the dashboards from Grafana Dashboards or by exporting them and place them in a directory together. Then create a template similar to this:

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-grafana-custom
  labels:
    app: grafana
    release: {{ .Release.Name }}
data:
  {{- range $path, $data := .Files.Glob "dashboards/*.json" }}
  {{ $path | base }}: |
    {{- $.Files.Get $path | nindent 4 }}
  {{- end }}

If you export a dashboard through the UI, it will be templated in such a way that it cannot be used as a provisioned dashboard. Instead, export using the HTTP API. It's also not very difficult to use simple find-and-replace to convert the variables (like ${DS_PROMETHEUS}) to literal values (like Prometheus). The literal value to use is the same as the data source name field in the datasources section below.

Then, in the grafana section of your values.yaml, add something like this. The dashboardProviders section should not need to be changed, only the dashboardsConfigMaps value.

dashboardsConfigMaps:
  default: myrelease-grafana-custom
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: "default"
        orgId: 1
        folder: ""
        type: file
        disableDeletion: false
        updateIntervalSeconds: 10
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

Alternately, you can paste dashboard JSON directly into the dashboards YAML block, but this can make your values.yaml very long.

Set up datasources for your Grafana instance. We can instruct Grafana to pull from the cluster-wide Prometheus instance as well as our application-local instance, so that we can use cluster metrics in our dashboards. It's important that the names of your data sources match the names of the data sources in the dashboard JSON files; otherwise, the dashboards will be empty.

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: "http://myrelease-prometheus:9090"
        access: proxy
        basicAuth: false
        isDefault: true
      - name: Cluster Prometheus
        type: prometheus
        url: "http://prometheus-prometheus-oper-prometheus.monitoring:9090"
        access: proxy
        basicAuth: false

As before, you can set adminPassword to a password of your choice to avoid having a random one created for you.
Custom Metrics

The Kubernetes API server allows the user to provide it with "custom metrics" for objects like pods and deployments in addition to the built-in processor and memory usage metrics. These metrics can then be used for pod autoscaling, and potentially for other future integrations. While any system exposing a Kubernetes API service can be used, the Prometheus Adapter makes it particularly easy to hook Prometheus up to the custom metrics API, exposing all pod-related Prometheus metrics as Kubernetes custom metrics. The steps below describe how to install the Prometheus Adapter using its Helm chart.

Integrating custom API services into Kubernetes requires appropriate settings on the Kubernetes control plane. These settings are enabled in recent versions of Kubernetes on IBM Cloud, but older clusters in IBM Cloud or elsewhere may not have the necessary settings to support API aggregation.

It's important to note that only one instance of the Prometheus Adapter can provide metrics to the API server at a time. There are proposals to support combining metrics from multiple sources in the Adapter or more generically, but since this is not possible right now, we will work around the issue by forwarding metrics from application-specific Prometheus instances to the cluster-wide instance.

To set up the Prometheus Custom Metrics Adapter:

    Edit the Prometheus Operator overrides to add another additional service monitor. If you followed the instructions above, this means adding another list item to additionalServiceMonitors in step 2 of Set Up Cluster-Wide Monitoring. This will instruct the cluster-wide Prometheus instance to pull metrics from other Prometheus instances in the cluster.

    # additionalServiceMonitors:
    - name: user-prometheus
      namespaceSelector:
        any: true
      selector:
        matchLabels:
          app: prometheus # Or whatever
      endpoints:
        - port: http
          path: /federate
          honorLabels: true
          params:
            match[]:
              - '{__name__=~".+"}'

    You will need to ensure that the label selector here matches all Prometheus instances whose metrics you want to aggregate. If there are some instances you want to ignore, you can use the selector to exclude them. It doesn't matter whether the selector matches the cluster-wide Prometheus itself.

    Upgrade or reinstall the Prometheus Operator chart to apply the new service monitor. Now, if you port-forward the cluster-wide Prometheus and visit http://localhost:9090/targets, you should see all other Prometheus instances listed under monitoring/user-prometheus/0. If not, check again that the labels match the service monitor's selector.

    To install the Prometheus Adapter, run the following Helm command:

    helm install --namespace monitoring -n prometheus-adapter stable/prometheus-adapter \
        --set prometheus.url="http://prometheus-prometheus-oper-prometheus" \
        --set metricsRelistInterval=2m

    The Prometheus URL should be consistent across installations, but you may want to check kubectl get svc -n monitoring just to make sure. The metricsRelistInterval setting will keep federated metrics from oscillating in and out of existence in the custom metrics API.

    Run kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1, and you should see a list of all Prometheus metrics as Kubernetes interprets them. It may take a couple minutes for the metrics to get all the way through the pipeline.

You can now create a horizontal pod autoscaler using custom metrics like this (example from the Global Air Quality proof of concept stack):

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-oapi-gaq
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: {{ .Release.Name }}-oapi-gaq
  minReplicas: 1
  maxReplicas: 8
  metrics:
    - type: Pods
      pods:
        metricName: sunCluster_metrics_requests_OneMinuteRate
        targetAverageValue: 50

In this case, the existence of the desired metric, sunCluster_metrics_requests_OneMinuteRate, can be confirmed by running kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/sunCluster_metrics_requests_OneMinuteRate. This will scale the deployment to as few as one or as many as eight pods, trying to minimize the number of pods while keeping the referenced metric below the targetAverageValue of 50.
Ingress Controller Monitoring

You can install the ALB Metrics Exporter in your cluster to get metrics from the ingress controller into your cluster-wide Prometheus instance. This will make available metrics like the number of various HTTP status codes, the backend latency, and the current number of open connections. Keep in mind that this won't give metrics on applications exposed on a host port, and it won't work if you set up your own ingress controller.

The ALB Metrics Exporter comes with its own Helm chart, but the ICM team has created an alternate chart to fix some shortcomings in the official version. Follow the instructions to get started with the ICM chart. This will give you a set of new metrics starting with alb_. You can view these metrics with a Grafana dashboard in that repository.
IBM Cloud Monitoring Service

    High level list of monitoring solutions
    Getting Started with IBM Cloud Monitoring provides an introduction and overview of how to setup monitoring and alerting.
    Provisioning the Monitoring Service describes how to create the monitoring service for your account via the UI or CLI.

IBM Cloud Monitoring Service as Solution
Sending Custom Metrics

    Sending data by using the Metrics API
    Sending data by using the Monitoring plugin (collectd)

Retrieving Metrics

    Retrieving metrics via the Metrics API

Container Service Monitoring

Monitoring IBM Cloud Container Service explains how to monitor container services, what metrics are available, how to work with Grafana, and what permissions are needed. Security for Cloud Monitoring explains the different roles in more detail.
Configuring Metrics in Grafana

    Configuring CPU metrics for a container in Grafana
    Configuring load metrics for a worker in Grafana
    Configuring memory metrics for a container in Grafana

