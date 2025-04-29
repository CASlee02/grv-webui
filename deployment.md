# GRV-WebUI Kubernetes Deployment Guide

This guide provides instructions for deploying the customized GRV-WebUI application to an Azure Kubernetes Service (AKS) cluster.

## Overview

This deployment configures the GRV-WebUI application, a modified version of Open WebUI. Key characteristics of this setup include:

*   **Custom Application Image**: Uses a custom Docker image (`csleetl.azurecr.io/grv-webui:latest`) hosted on Azure Container Registry (ACR).
*   **External LLM**: Relies on an external Large Language Model service (configuration details depend on the specific service). Ollama is **not** deployed as part of this setup.
*   **External Database**: Connects to an external PostgreSQL database for persistence.
*   **Azure Integration**:
    *   Uses AKS Managed Identity for seamless authentication with ACR.
    *   Uses Azure Workload Identity for potentially accessing other Azure resources securely from within the pods.
*   **Kubernetes Manifests**: Deployment is managed via standard Kubernetes manifest files located in `kubernetes/manifest/base`.

## Prerequisites

*   **kubectl**: Kubernetes command-line tool installed and configured to connect to your target AKS cluster.
*   **Azure CLI**: Azure command-line tool installed and logged in with permissions to manage AKS and ACR.
*   **AKS Cluster**: An existing Azure Kubernetes Service cluster.
*   **Azure Container Registry (ACR)**: An ACR instance (`csleetl.azurecr.io`) hosting the custom application image (`grv-webui:latest`).
*   **External PostgreSQL Database**: An accessible PostgreSQL database instance.
*   **Permissions**: Necessary permissions to create namespaces, deployments, services, secrets, service accounts, and potentially configure Workload Identity federation in the AKS cluster.

## Kubernetes Components

The deployment consists of the following Kubernetes resources defined in `kubernetes/manifest/base/`:

1.  **Namespace**: `grv-webui` - A dedicated namespace for all application resources.
2.  **Deployment (`webui-deployment.yaml`)**:
    *   Manages the application pods using the custom image `csleetl.azurecr.io/grv-webui:latest`.
    *   Configured to use Azure Workload Identity via the `openwebui-sa` service account.
    *   Sets environment variables for database connection (`DATABASE_URL` from secret, pool size, etc.).
    *   Specifies resource requests and limits for the application container.
    *   Uses a Persistent Volume Claim (`open-webui-pvc`) for data storage (though primary data is in the external DB).
3.  **Service (`webui-service.yaml`)**:
    *   Exposes the application externally using a `LoadBalancer` service type on port 80.
    *   Routes traffic to the application pods on port 8080.
4.  **Persistent Volume Claim (`webui-pvc.yaml`)**:
    *   Requests persistent storage (2Gi) for application data mounted at `/app/backend/data`.
5.  **Secret (`db-credentials` - Manual Creation)**:
    *   Stores the database connection string (`database-url`). **Must be created manually.**
6.  **Service Account (`openwebui-sa` - Manual or via Manifest)**:
    *   Used by the deployment pods for Azure Workload Identity. **Must be created manually or via a separate manifest, and federated identity configured.**

## Deployment Steps

1.  **Connect to AKS Cluster**: Ensure `kubectl` is configured to point to your target AKS cluster.
    ```bash
    # Example: Get credentials for your AKS cluster
    az aks get-credentials --resource-group service_knowledge_app --name GRVWebUI
    ```

2.  **Connect AKS to ACR**: Grant the AKS cluster permission to pull images from your ACR instance using managed identity.
    ```bash
    az aks update --name <your-aks-cluster-name> --resource-group <your-resource-group> --attach-acr csleetl.azurecr.io
    ```

3.  **Create Namespace**:
    ```bash
    kubectl create namespace grv-webui
    ```

4.  **Create Database Secret**: Create the Kubernetes secret containing the database connection URL. Replace `<your-database-url>` with the actual connection string.
    ```bash
    kubectl create secret generic db-credentials --from-literal=database-url='<your-database-url>' -n grv-webui
    ```

5.  **Configure Workload Identity (If not already set up)**:
    *   Create the Kubernetes Service Account:
        ```bash
        kubectl create serviceaccount openwebui-sa -n grv-webui
        ```
    *   Set up the Federated Identity Credential between the Azure AD Application/Managed Identity and the Kubernetes Service Account. Refer to Azure documentation for setting up Workload Identity federation.

6.  **Apply Manifests**: Apply the Kubernetes resources defined in the `manifest/base` directory.
    ```bash
    # Navigate to the repository root or adjust paths accordingly
    kubectl apply -f kubernetes/manifest/base/webui-pvc.yaml -n grv-webui
    kubectl apply -f kubernetes/manifest/base/webui-deployment.yaml -n grv-webui
    kubectl apply -f kubernetes/manifest/base/webui-service.yaml -n grv-webui
    ```

7.  **Verify Deployment**:
    *   Check pod status:
        ```bash
        kubectl get pods -n grv-webui -w
        ```
        *(Wait for the `open-webui-deployment-xxxxx` pod to be `Running`)*
    *   Get the external IP address of the service:
        ```bash
        kubectl get service open-webui-service -n grv-webui
        ```
        *(Look for the `EXTERNAL-IP`)*

8.  **Access Application**: Open a web browser and navigate to `http://<EXTERNAL-IP>`.

## Updating the Application

1.  **Build and Push New Image**: Build your updated application code and push the new Docker image to ACR with the `latest` tag (or a new version tag).
    ```bash
    docker build -t csleetl.azurecr.io/grv-webui:latest .
    docker push csleetl.azurecr.io/grv-webui:latest
    ```
    *(If using a new version tag, update `kubernetes/manifest/base/webui-deployment.yaml` accordingly)*

2.  **Rollout Restart**: Trigger a rolling update of the deployment to pull the new image.
    ```bash
    kubectl rollout restart deployment open-webui-deployment -n grv-webui
    ```

3.  **Monitor Rollout**:
    ```bash
    kubectl rollout status deployment open-webui-deployment -n grv-webui
    ```

## Troubleshooting

*   **ImagePullBackOff**: Verify AKS has permissions to ACR (`az aks update --attach-acr`). Check ACR name and image tag in `webui-deployment.yaml`. Ensure the image exists in ACR.
*   **CrashLoopBackOff**: Check pod logs (`kubectl logs <pod-name> -n grv-webui`). Verify database credentials and connectivity. Ensure environment variables are correctly set. Check Workload Identity configuration if the application relies on it to access Azure resources.
*   **Pending Pods**: Check resource availability (`kubectl describe node <node-name>`). Ensure PVC can be bound (`kubectl get pvc -n grv-webui`).
*   **Service External IP Pending**: Ensure your AKS cluster has LoadBalancer services enabled and quota available.