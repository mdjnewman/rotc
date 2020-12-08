# 04-01 Metrics collection and visualisation

You will learn about:

* application instrumentation / metric collection with Prometheus
* metrics visualisation in Grafana dashboards

## Start

In the start state, you are provided with a version 3 of the `DockerCoins` web UI code as well as its Kubernetes YAML deployment manifest.

Run `cd exercise/` and follow the instructions below to get started!

## Application instrumentation with Prometheus

Instrumentation is a term that refers to:

* adding application logic, generally using a client library
* that will expose application metrics, typical user-defined metrics and performance metrics that will help to diagnose or trace activity.

### Adding metrics to a NodeJS application

We will use the `DockerCoins` webui application for this part. This version 3 of the webui adds an endpoint `/metrics` to the existing HTTP API.

Run the webui locally with the following commands:

```console
npm install
npm run start
```

Redis will throw errors as it is not running but the webui should be still be accessible on port `80`. Generate now traffic on the application by accessing the following HTTP endpoints:

* <http://localhost/info>
* <http://localhost/index.html>

Looking at the `webui.js` file, you will find where the instrumentation code sits:

```js
var swStats = require('swagger-stats');
...
app.use(swStats.getMiddleware());
...
```

We've used here a NodeJS middleware provided by the client library [swagger-stats](http://swaggerstats.io/docs.html). swagger-stats effectively creates the `/metrics` endpoint and expose NodeJS metrics in a format that is already digestabble by Prometheus.

If you work with another language (C#, Java, Python), Prometheus provides different client libraries for the main languages. See [Prometheus client libraries](https://prometheus.io/docs/instrumenting/clientlibs/).

Go to the `/metrics` endpoint and look at the metrics available: `http://localhost/swagger-stats/metrics`.
Some of these metrics exposed by swagger stats will be useful to do monitoring via Prometheus. For example:

* `api_all_success_total` is the number of successful requests on the application
* `api_request_duration_milliseconds_bucket` represents the duration of each request

If you want a visual representation of these metrics go to the swagger-stats dashboard located at <http://localhost/swagger-stats/ui>

At this stage we've instrumented our application. From here, Prometheus will be able to then scrape this endpoint and collect metrics from our application.

### Collecting application metrics with Prometheus in Kubernetes

To enable Prometheus' scraping on our new API endpoint, we need to use annotations in the Kubernetes YAML deployment definition. Prometheus needs it to scrape the pods running our application and get the metrics from the newly creaated `/metrics` endpoint.

Let's look at these annotations in the YAML definition of our deployment (see file `prometheus-app.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dockercoins
spec:
  selector:
    matchLabels:
      app: dockercoins
  template:
    metadata:
      labels:
        app: dockercoins
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: 'swagger-stats/metrics'
    spec:
      containers:
      - # ...
```

Prometheus leverages the Kubernetes APIs to dynamically detects pods that expose metrics based on the presence of the `prometheus.io/*` annotations.

The per-pod Prometheus annotations available are:

* `prometheus.io/scrape`: Enable or disable Prometheus scraping
* `prometheus.io/path`: HTTP path of the metrics endpoints (default is `/metrics`)
* `prometheus.io/port`: Use a scraping port different to the port declared by the pod

Start the deployment with:

```console
kubectl apply -f prometheus-app.yaml
```

Verify that `DockerCoins` webui version is up:

```console
kubectl get pods
```

After a few seconds/minutes, the output should be similar to (5 services of 5 running):

```output
NAME                        READY   STATUS    RESTARTS   AGE
dockercoins-6b849c9888-t2js4   5/5     Running   0          2m4s
```

Access the webui with:

```console
kubectl port-forward $(kubectl get pod -l app=dockercoins -o jsonpath='{.items[0].metadata.name}') 3000:80
```

And visit a few endpoints:

* <http://localhost:3000/info>
* <http://localhost:3000/>

### Visualise metrics in the Prometheus expression browser

The Prometheus expression browser is a web ui provided by Prometheus itself showing all the metrics collected.

Start it by forwarding the local port `9090` on your client machine to port `9090` on the pod that is running Prometheus in your Kubernetes cluster:

```console
kubectl -n prometheus port-forward $(kubectl -n prometheus get pods -o jsonpath='{.items[0].metadata.name}') 9090:9090
```

Go to <http://localhost:9090>.

Search for the expression `api_all_success_total` in the search bar and execute! You should see a the total number of HTTP requests executed on the `DockerCoins` webui :)

Prometheus is a very powerful tool which with PromQL provides advanced querying features. We don't cover these features here but a good starting point to learn about it is look at the [Prometheus data types](https://prometheus.io/docs/prometheus/latest/querying/basics/).

Links:

* [Tools for Monitoring Resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)

### Grafana application dashboard

Now that our application metrics are stored in Prometheus, we can use Grafana to visualise them.

Start the grafana web ui with:

```console
kubectl -n grafana port-forward $(kubectl -n grafana get pod -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
```

The dashboard should now be accessible at <http://localhost:3000>.

From here, select the dashboard called "swagger-stats dashboard release".

You should now see live metrics coming from Prometheus!

In this section, we've send metrics from our application to Prometheus and graph them in Grafana.

Grafana also integrates with Prometheus Alertmanager. If issues arise or new users patterns are found based on metrics' thresholds,  alerts can be configured to send SMS/chat messages to all the developers in the team that are looking at support activities.

### Resources

* [Kubernetes grafana dashboard](https://github.com/Thakurvaibhav/k8s/tree/master/monitoring/dashboards)
* [Istio grafana dashboards](https://istio.io/docs/tasks/observability/metrics/using-istio-dashboard/)

## Cleanup

```console
# Windows only
kubectl delete all --all -n "$env:TEAM_NAME"

# MacOS
kubectl delete all --all -n "${TEAM_NAME}"
```
