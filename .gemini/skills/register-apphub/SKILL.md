---
name: register-apphub
description: >-
  Registers the deployed OpenTelemetry Demo with App Hub. Use when you need to
  enable App Hub integration for the demo, register its GKE workloads and
  services, or set up App Topology and Monitoring.
---

# Register OpenTelemetry Demo with App Hub

This skill guides you through registering the OpenTelemetry Demo (already deployed to GKE) with App Hub to enable App Topology and Monitoring.

## Prerequisites

*   The OpenTelemetry Demo must be already deployed to a GKE cluster.
*   GKE Managed OpenTelemetry should be enabled on the cluster (if you want automatic telemetry association).

## Step-by-Step Instructions

### Step 1: Gather User Inputs

Ask the user for the following information before running any commands:
1.  **Project ID**: The Google Cloud Project where the demo is deployed.
2.  **Cluster Name**: The GKE cluster name.
3.  **Cluster Location**: The region or zone of the GKE cluster.
4.  **Application Name**: The desired name for the App Hub application (default: `otel-demo-app`).
5.  **User Email**: The email of the owner/developer (for App Hub attributes).

### Step 2: Enable Required APIs

Enable the App Hub and App Topology APIs in the specified project:

```bash
gcloud services enable \
    apphub.googleapis.com \
    apptopology.googleapis.com \
    --project={project_id}
```

### Step 3: Create App Hub Application

Create a regional App Hub application. Use the cluster location as the application location:

```bash
gcloud apphub applications create {application_name} \
    --location={cluster_location} \
    --project={project_id} \
    --display-name="OpenTelemetry Demo Application" \
    --scope-type=REGIONAL \
    --environment-type=PRODUCTION \
    --criticality-type=HIGH \
    --developer-owners=email={user_email}
```

### Step 4: Register Discovered GKE Workloads

Register all workloads in the `otel-demo` namespace to the application:

```bash
gcloud apphub discovered-workloads list \
  --location={cluster_location} \
  --project={project_id} \
  --filter="workloadReference.uri:namespaces/otel-demo" \
  --format="value(name, workloadReference.uri)" | while read -r name uri; do
    workload_id=$(basename "$name")
    workload_name=$(basename "$uri")
    echo "Registering Workload: $workload_name ($workload_id)"
    gcloud apphub applications workloads create "$workload_name" \
      --application={application_name} \
      --location={cluster_location} \
      --discovered-workload="$workload_id" \
      --project={project_id} \
      --environment-type=PRODUCTION
done
```

### Step 5: Register Discovered GKE Services

Register all GKE services in the `otel-demo` namespace:

```bash
gcloud apphub discovered-services list \
  --location={cluster_location} \
  --project={project_id} \
  --filter="serviceReference.uri:namespaces/otel-demo" \
  --format="value(name, serviceReference.uri)" | while read -r name uri; do
    service_id=$(basename "$name")
    service_name=$(basename "$uri")
    echo "Registering Service: $service_name ($service_id)"
    gcloud apphub applications services create "$service_name" \
      --application={application_name} \
      --location={cluster_location} \
      --discovered-service="$service_id" \
      --project={project_id}
done
```

### Step 6: Register External Load Balancer Resources

Register the GCE Load Balancer resources (Backend Service and Forwarding Rule) associated with the `frontend-proxy` GKE service.

1.  Find the GKE Service UID for `frontend-proxy`:
    ```bash
    gcloud container clusters get-credentials {cluster_name} --region {cluster_location} --project {project_id}
    kubectl get svc frontend-proxy -n otel-demo -o jsonpath='{.metadata.uid}'
    ```
    *Note the output UID (e.g., `02fee0ca-813e-441e-a3a2-ab819c48ec11`). Remove dashes to get the resource suffix (e.g., `02fee0ca813e441ea3a2ab819c48ec11`).*

2.  List discovered services and find the IDs for the Backend Service and Forwarding Rule matching that suffix:
    ```bash
    gcloud apphub discovered-services list \
        --location={cluster_location} \
        --project={project_id} \
        --filter="serviceReference.uri:{suffix}"
    ```
3.  Register them:
    ```bash
    # Register Backend Service
    gcloud apphub applications services create frontend-proxy-lb-backend \
        --application={application_name} \
        --location={cluster_location} \
        --discovered-service={backend_service_discovered_id} \
        --project={project_id} \
        --display-name="Frontend Proxy LB Backend"

    # Register Forwarding Rule
    gcloud apphub applications services create frontend-proxy-lb-forwarding-rule \
        --application={application_name} \
        --location={cluster_location} \
        --discovered-service={forwarding_rule_discovered_id} \
        --project={project_id} \
        --display-name="Frontend Proxy LB Forwarding Rule"
    ```

### Step 7: Provide Console Links to User

Provide the user with direct links to verify the integration:

*   **App Hub Registry**: `https://pantheon.corp.google.com/apphub?project={project_id}`
*   **Application Monitoring Dashboard**: `https://pantheon.corp.google.com/monitoring/applications/{cluster_location}/{application_name}/dashboard?project={project_id}`
*   **App Topology View**: `https://pantheon.corp.google.com/monitoring/applications/{cluster_location}/{application_name}/topology?project={project_id}`
