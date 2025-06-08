# Cluster Setup Guide

This guide describes how to set up a local Kubernetes cluster using **Kind (Kubernetes IN Docker)** for development and testing purposes with this GitOps repository.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (Ensure Docker Desktop or Docker Engine is running)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Creating a Kind Cluster

We will create a multi-node Kind cluster to better simulate a real environment, though a single-node cluster also works for many scenarios.

1.  **Create a Kind Cluster Configuration File:**
    Create a file named `kind-cluster-config.yaml` with the following content:

    ```yaml
    # kind-cluster-config.yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    name: gitops-cluster # You can name your cluster
    nodes:
    - role: control-plane
      # (Optional) Port mappings for ingress controller if you plan to expose services via NodePort
      # kubeadmConfigPatches:
      # - |
      #   kind: InitConfiguration
      #   nodeRegistration:
      #     kubeletExtraArgs:
      #       node-labels: "ingress-ready=true"
      # extraPortMappings:
      # - containerPort: 80
      #   hostPort: 80
      #   protocol: TCP
      # - containerPort: 443
      #   hostPort: 443
      #   protocol: TCP
    - role: worker
    - role: worker
    ```
    *Note: Adjust `extraPortMappings` if you plan to use an Ingress controller like Nginx and want to access it via `localhost:80` or `localhost:443`.*

2.  **Create the Cluster:**
    Run the following command in your terminal in the same directory as `kind-cluster-config.yaml`:

    ```bash
    kind create cluster --config ./kind-cluster-config.yaml
    ```
    This command will download the necessary node images (if not already present) and bootstrap your Kubernetes cluster. It might take a few minutes.

3.  **Verify Cluster Access:**
    Once the cluster is created, Kind automatically configures `kubectl` to use the new cluster context. You can verify this by running:

    ```bash
    kubectl cluster-info --context kind-gitops-cluster
    kubectl get nodes
    ```
    You should see information about your cluster and a list of nodes (1 control-plane, 2 workers).

## Next Steps

With your Kind cluster up and running, you can now proceed to:

1.  **Install ArgoCD:** Follow the instructions in `bootstrap/README.md` (to be created) or the main [README.md](./README.md).
2.  **Deploy OLM (Operator Lifecycle Manager):** OLM is a prerequisite for installing operators. The ArgoCD application for shared resources will typically handle this.
    *(We might add a manual OLM installation step here later if needed before ArgoCD takes over, or link to a script in `scripts/setup/`)*

## Deleting the Cluster

When you are finished with the cluster, you can delete it using:

```bash
kind delete cluster --name gitops-cluster
```
