# Troubleshooting Guide

This guide provides solutions and tips for common issues encountered while working with this GitOps repository, ArgoCD, and Kubernetes.

## ArgoCD Issues

**1. Application OutOfSync / Sync Failed**
   - **Symptoms:** ArgoCD UI shows the application as `OutOfSync` or `SyncFailed`.
   - **Troubleshooting:**
     - Check the sync error details in the ArgoCD UI for the specific application. This often points to a malformed manifest, a missing CRD, or an issue applying a resource.
     - Verify that the Git commit ArgoCD is trying to sync is correct and accessible.
     - Use `argocd app diff <app-name>` to see what ArgoCD perceives as different from the live state.
     - Check ArgoCD controller logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`
     - **Common Cause (Operators):** If deploying a Custom Resource (CR) before its CRD (managed by an operator) is fully established, the sync will fail. Ensure operators are healthy and CRDs are available. The "App of Apps" structure with ordered syncs (e.g., `shared-resources-app` before `applications-app`) helps mitigate this.

**2. ImagePullBackOff / ErrImagePull**
   - **Symptoms:** Pods are stuck in `ImagePullBackOff` or `ErrImagePull` state.
   - **Troubleshooting:**
     - `kubectl describe pod <pod-name> -n <namespace>` will show the exact error.
     - **Incorrect Image Name/Tag:** Verify the image name and tag in your `Deployment` or `StatefulSet` manifest are correct and exist in the container registry.
     - **Registry Authentication:** If pulling from a private registry, ensure an `imagePullSecret` is correctly configured and referenced in your Pod spec or ServiceAccount.
     - **Network Issues:** Ensure the Kubernetes nodes can reach the container registry.

**3. CrashLoopBackOff**
   - **Symptoms:** Pods are repeatedly starting and crashing.
   - **Troubleshooting:**
     - `kubectl logs <pod-name> -n <namespace>`: Check the application logs for errors (e.g., configuration issues, database connection failures, startup exceptions).
     - `kubectl logs <pod-name> --previous -n <namespace>`: Check logs from the previously crashed container.
     - **Configuration Errors:** Double-check `ConfigMap` and `Secret` values. Ensure environment variables are correctly set.
     - **Resource Limits:** The application might be crashing due to insufficient CPU or memory. Check resource requests and limits.
     - **Liveness/Readiness Probes:** Misconfigured probes can cause pods to be restarted unnecessarily. Verify probe endpoints, timeouts, and thresholds.

**4. CRD Not Found**
   - **Symptoms:** ArgoCD fails to sync an application, complaining that a Custom Resource Definition (CRD) is not found.
   - **Troubleshooting:**
     - **Operator Not Installed/Healthy:** Ensure the operator responsible for the CRD is installed and running correctly. Check the operator's logs.
     - **OLM Subscription Issues:** If using OLM, check the status of the `Subscription` and `InstallPlan` for the operator in the `operators` (or relevant) namespace.
     - **Sync Order:** The ArgoCD application managing the operator (e.g., via `shared-resources-app`) must sync *before* the application attempting to use the CRD.

## Kubernetes / Application Issues

**1. Service Not Accessible / Connection Refused**
   - **Symptoms:** Unable to connect to an application's service.
   - **Troubleshooting:**
     - **Service Selector:** Ensure the `selector` in your `Service` manifest correctly matches the `labels` on your `Pods`. Use `kubectl get pods -n <namespace> -l <key>=<value>`.
     - **Target Port:** Verify the `targetPort` in the `Service` matches the `containerPort` your application is listening on inside the Pod.
     - **Endpoints:** Check if the Service has active endpoints: `kubectl get endpoints <service-name> -n <namespace>`. If empty, there's likely a selector mismatch or pods are not ready.
     - **Network Policies:** If `NetworkPolicy` resources are in use, ensure they allow traffic to the service from the source you are testing from.
     - **Ingress Configuration:** If accessing via Ingress, check Ingress controller logs and the Ingress resource definition.

**2. PersistentVolumeClaim (PVC) Pending**
   - **Symptoms:** `StatefulSet` or `Deployment` pods are stuck in `Pending` because their PVCs are `Pending`.
   - **Troubleshooting:**
     - `kubectl describe pvc <pvc-name> -n <namespace>`: Check events for reasons.
     - **StorageClass Issues:** Ensure a default `StorageClass` is available, or that the PVC specifies a valid, existing `StorageClass`.
     - **Insufficient Resources:** The underlying storage provisioner might not have enough capacity.
     - **Node Affinity/Selector Issues:** If using local persistent volumes, ensure the pod can be scheduled on a node that can provide the volume.

## SOPS / Secrets Issues

**1. Decryption Failed**
   - **Symptoms:** ArgoCD (if configured with a SOPS plugin) or a manual `sops -d` command fails to decrypt a secret.
   - **Troubleshooting:**
     - **GPG Key Not Available:** Ensure the GPG key used for encryption is available in the GPG keychain of the user/system trying to decrypt (or in ArgoCD's decryption mechanism).
     - **Incorrect Key:** The file might have been encrypted with a different GPG key.
     - **Corrupted File:** The `.sops.yaml` file might be corrupted.
     - **Cloud KMS Permissions:** If using AWS KMS, GCP KMS, or Azure Key Vault, ensure the IAM role/principal has decryption permissions.

## General Debugging Tips

-   **Start Small:** If deploying a complex set of applications, try deploying them one by one to isolate issues.
-   **Kustomize Build:** Locally run `kustomize build <overlay_path>` to inspect the final YAML that will be applied to the cluster. This can help catch templating or patching errors.
-   **`kubectl diff -f <file_or_directory>`:** Before applying, see what changes `kubectl apply` would make.
-   **Check Events:** `kubectl get events -n <namespace> --sort-by='.lastTimestamp'` can provide valuable clues about what's happening in the cluster.

