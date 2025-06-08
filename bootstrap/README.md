# Bootstrap ArgoCD Installation

This guide explains how to install ArgoCD into your Kubernetes cluster and configure it to manage resources from this GitOps repository.

## 1. Install ArgoCD

We recommend installing ArgoCD using the official installation manifests provided by the ArgoCD project.

1.  **Create the ArgoCD Namespace:**
    ```bash
    kubectl create namespace argocd
    ```

2.  **Apply the Installation Manifest:**
    Choose one of the following installation types. For most local development (like with Kind) and basic setups, the non-HA manifest is sufficient.

    *   **Non-HA (High Availability):**
        ```bash
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
        ```

    *   **HA (High Availability):**
        For production or more resilient setups, use the HA manifests:
        ```bash
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
        ```
    *Note: Always check the [ArgoCD Getting Started Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/) for the latest stable manifest URL.*

3.  **Verify Installation:**
    Wait for the ArgoCD pods to be up and running:
    ```bash
    kubectl get pods -n argocd -w
    ```

## 2. Access ArgoCD

*   **Via UI (Port Forwarding for Local Access):**
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
    Then open `https://localhost:8080` in your browser.

*   **Get Initial Admin Password:**
    The initial admin password is auto-generated and stored in a secret:
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
    Login with username `admin` and the retrieved password. It's highly recommended to change this password after your first login.

*   **Via CLI:**
    Install the [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/). Then login:
    ```bash
    argocd login localhost:8080 --username admin --password <your-retrieved-password> --insecure
    ```

## 3. Apply the Root Application for Your Cluster

Once ArgoCD is installed and accessible, you need to apply the "root" ArgoCD `Application` CRD that corresponds to your target cluster. This root application will then manage all other ArgoCD applications and resources for that cluster defined in this repository.

For example, to bootstrap the **development cluster**:

1.  Ensure your `kubectl` context is pointing to your **development Kubernetes cluster**.
2.  Apply the development root application:
    ```bash
    # Ensure this path points to the correct root-app for your target cluster.
    # The 'destination.server' in this root-app.yaml should also point to the correct cluster.
    # For a single ArgoCD instance managing multiple clusters, the destination.name or destination.server
    # will be critical. For an ArgoCD instance per cluster, destination.server is usually kubernetes.default.svc.

    kubectl apply -f ../clusters/development/bootstrap/root-app.yaml -n argocd
    ```
    *Replace `../clusters/development/bootstrap/root-app.yaml` with the path to the `root-app.yaml` for `staging` or `production` clusters as needed, ensuring your `kubectl` context is set to that respective cluster.*

ArgoCD will now start syncing the resources defined by this root application, which in turn will create other ArgoCD applications for shared resources and workloads.

## (Optional) Configure Repository Access

If your Git repository is private, you will need to configure ArgoCD with access credentials. This can be done via the ArgoCD UI (Settings -> Repositories) or by creating a Kubernetes secret and an ArgoCD `Repository` CRD.

Refer to the [ArgoCD documentation on Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/) for detailed instructions.
