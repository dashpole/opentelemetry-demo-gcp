# Running the demo on Google Cloud

The demo can send logs, traces, and metrics to Google Cloud. The easiest way to
do this is with the [`gcp/helmfile.yaml`](gcp/helmfile.yaml) (when
deploying via Helmfile on GKE) or
[`src/otelcollector/otelcol-config-extras.yml`](src/otelcollector/otelcol-config-extras.yml)
when running with `docker-compose` on GCE.

## Running on GKE

The recommended way to run the demo on GKE is with the official Helm chart, configured to use GKE-managed OpenTelemetry.

### Prerequisites

You must enable Managed OpenTelemetry on your GKE cluster (requires GKE version >= 1.34.1-gke.2178000).

For a new Autopilot cluster, you can enable it during creation:
```console
gcloud beta container clusters create-auto <cluster-name> \
    --managed-otel-scope=COLLECTION_AND_INSTRUMENTATION_COMPONENTS \
    --location=<location>
```
For an existing cluster, you can enable it using:
```console
gcloud beta container clusters update <cluster-name> \
    --managed-otel-scope=COLLECTION_AND_INSTRUMENTATION_COMPONENTS \
    --location=<location>
```

See [Managed OpenTelemetry for GKE documentation](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/managed-otel-gke) for more details.


### Deploying the Helmfile

Make sure you have the following installed:

* [Helmfile](https://helmfile.readthedocs.io/en/stable/#installation)
* [Helm](https://helm.sh/docs/intro/install/)
* [helm-diff plugin](https://github.com/databus23/helm-diff)

Ensure you have either configured or disabled Workload Identity in your cluster.

```console
helmfile --interactive apply -f gcp/helmfile.yaml
```

#### Cleaning up the helmfile

To clean up, run:

```console
helmfile --interactive destroy -f gcp/helmfile.yaml
```

### (Alternative) Using `kubectl apply`

Installing with the Helm chart is recommended, but you can also use `kubectl
apply` to install the manifests directly.

First, make sure you have followed the Workload Identity setup steps above.

Install the manifests:

```console
kubectl apply -n otel-demo -f ./kubernetes/opentelemetry-demo.yaml
```

## Running on GCE

Follow the [OpenTelemetry docs to run with Docker](https://opentelemetry.io/docs/demo/docker-deployment/):

```console
make start
```

## Seeing telemetry

With the demo running, you should see telemetry automatically created by the
demo's load generator. You can see metrics under "Prometheus Target" in Cloud
Monitoring:

![metrics](gcp_metrics.png)

Traces in the Trace explorer:

![traces](gcp_traces.png)

And logs in the Logs explorer organized by service:

![logs](gcp_logs.png)
