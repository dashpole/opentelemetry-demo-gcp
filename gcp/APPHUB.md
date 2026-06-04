# Registering OpenTelemetry Demo with App Hub

This guide describes the steps necessary to register the OpenTelemetry Demo (deployed to GKE) with Google Cloud App Hub to enable App Topology and Monitoring.

## Prerequisites

*   The OpenTelemetry Demo must be deployed to a GKE cluster.
*   GKE Managed OpenTelemetry should be enabled on the cluster for automatic telemetry association.

## Step-by-Step Instructions

### Step 1: Define Variables

Identify the following variables for your deployment:

*   `PROJECT_ID`: The Google Cloud Project ID where the demo is deployed.
*   `CLUSTER_NAME`: The GKE cluster name.
*   `CLUSTER_LOCATION`: The region or zone of the GKE cluster (e.g., `us-central1`).
*   `APPLICATION_NAME`: The desired name for the App Hub application (e.g., `otel-demo-app`).
*   `USER_EMAIL`: Your email address (registered as the developer owner).

Set these in your terminal:

```bash
export PROJECT_ID="your-project-id"
export CLUSTER_NAME="your-cluster-name"
export CLUSTER_LOCATION="your-cluster-location"
export APPLICATION_NAME="otel-demo-app"
export USER_EMAIL="your-email@google.com"
```

### Step 2: Enable Required APIs

Enable the App Hub and App Topology APIs in your project:

```bash
gcloud services enable \
    apphub.googleapis.com \
    apptopology.googleapis.com \
    --project=${PROJECT_ID}
```

### Step 3: Create App Hub Application

Create a regional App Hub application:

```bash
gcloud apphub applications create ${APPLICATION_NAME} \
    --location=${CLUSTER_LOCATION} \
    --project=${PROJECT_ID} \
    --display-name="OpenTelemetry Demo Application" \
    --scope-type=REGIONAL \
    --environment-type=PRODUCTION \
    --criticality-type=HIGH \
    --developer-owners=email=${USER_EMAIL}
```

### Step 4: Register Discovered GKE Workloads

Register all workloads in the `otel-demo` namespace to the application:

```bash
gcloud apphub discovered-workloads list \
  --location=${CLUSTER_LOCATION} \
  --project=${PROJECT_ID} \
  --filter="workloadReference.uri:namespaces/otel-demo" \
  --format="value(name, workloadReference.uri)" | while read -r name uri; do
    workload_id=$(basename "$name")
    workload_name=$(basename "$uri")
    echo "Registering Workload: $workload_name ($workload_id)"
    gcloud apphub applications workloads create "$workload_name" \
      --application=${APPLICATION_NAME} \
      --location=${CLUSTER_LOCATION} \
      --discovered-workload="$workload_id" \
      --project=${PROJECT_ID} \
      --environment-type=PRODUCTION
done
```

### Step 5: Register Discovered GKE Services

Register all GKE services in the `otel-demo` namespace:

```bash
gcloud apphub discovered-services list \
  --location=${CLUSTER_LOCATION} \
  --project=${PROJECT_ID} \
  --filter="serviceReference.uri:namespaces/otel-demo" \
  --format="value(name, serviceReference.uri)" | while read -r name uri; do
    service_id=$(basename "$name")
    service_name=$(basename "$uri")
    echo "Registering Service: $service_name ($service_id)"
    gcloud apphub applications services create "$service_name" \
      --application=${APPLICATION_NAME} \
      --location=${CLUSTER_LOCATION} \
      --discovered-service="$service_id" \
      --project=${PROJECT_ID}
done
```

### Step 6: Register External Load Balancer Resources

Register the GCE Load Balancer resources (Backend Service and Forwarding Rule) associated with the `frontend-proxy` GKE service.

1.  Get the GKE Service UID for `frontend-proxy`:
    ```bash
    gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${CLUSTER_LOCATION} --project ${PROJECT_ID}
    SVC_UID=$(kubectl get svc frontend-proxy -n otel-demo -o jsonpath='{.metadata.uid}')
    # Remove dashes to match App Hub discovery resource suffix
    SUFFIX=$(echo $SVC_UID | tr -d '-')
    ```

2.  List discovered services and find the IDs for the Backend Service and Forwarding Rule matching that suffix:
    ```bash
    gcloud apphub discovered-services list \
        --location=${CLUSTER_LOCATION} \
        --project=${PROJECT_ID} \
        --filter="serviceReference.uri:${SUFFIX}"
    ```
    *Note the full IDs for backend service and forwarding rule from the output.*

3.  Register them (replace placeholders with the actual discovered IDs from the previous step):
    ```bash
    BACKEND_SERVICE_DISCOVERED_ID="your-discovered-backend-service-id"
    FORWARDING_RULE_DISCOVERED_ID="your-discovered-forwarding-rule-id"

    # Register Backend Service
    gcloud apphub applications services create frontend-proxy-lb-backend \
        --application=${APPLICATION_NAME} \
        --location=${CLUSTER_LOCATION} \
        --discovered-service=${BACKEND_SERVICE_DISCOVERED_ID} \
        --project=${PROJECT_ID} \
        --display-name="Frontend Proxy LB Backend"

    # Register Forwarding Rule
    gcloud apphub applications services create frontend-proxy-lb-forwarding-rule \
        --application=${APPLICATION_NAME} \
        --location=${CLUSTER_LOCATION} \
        --discovered-service=${FORWARDING_RULE_DISCOVERED_ID} \
        --project=${PROJECT_ID} \
        --display-name="Frontend Proxy LB Forwarding Rule"
    ```

### Step 7: Verify Integration

You can view and verify the registered application and resources in the Google Cloud Console:

*   **App Hub Registry**: `https://console.cloud.google.com/apphub?project=${PROJECT_ID}`
*   **Application Monitoring Dashboard**: `https://console.cloud.google.com/monitoring/applications/${CLUSTER_LOCATION}/${APPLICATION_NAME}/dashboard?project=${PROJECT_ID}`
*   **App Topology View**: `https://console.cloud.google.com/monitoring/applications/${CLUSTER_LOCATION}/${APPLICATION_NAME}/topology?project=${PROJECT_ID}`
