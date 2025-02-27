# Installation and Clean up on Non-Kind Clusters

Although this project currently defaults to installing and running on a local Kind cluster, you can deploy k0rdent to other Kubernetes distributions. The following steps outline considerations and manual procedures required for non-Kind environments.

1. [Prerequisites and Caveats](#1-prerequisites-and-caveats)
1. [Installing k0rdent](#installing-k0rdent)
1. [Clean Up on a Non-Kind Cluster](#clean-up-on-a-non-kind-cluster)

## **Prerequisites and Caveats**

- **Flux CD Conflict**

    If Flux CD is already installed in your cluster, k0rdent will not install correctly. You must either uninstall Flux CD or install k0rdent in a separate cluster that does not have Flux CD.
        
- **Cert-Manager Conflict**

    If your cluster already has Cert-Manager installed (for example, from a previous installation or from another operator), you can disable k0rdent’s built-in Cert-Manager by adding the following Helm flag during installation:
   ```
   --set cert-manager.enabled=false
   ```

    This will instruct k0rdent not to install its own Cert-Manager components.

- **Helm Registry Port**

    By default, the Makefile sets:
    ```
    HELM_REGISTRY_EXTERNAL_PORT ?= 30500
    ```

    This is the NodePort used to expose the Helm registry inside your cluster. Many managed Kubernetes environments (e.g., Mirantis Kubernetes Engine (MKE)) may not allow using ports below 32500 by default. If you need a higher port range, override this with an environment variable before invoking make:
    ```
    export HELM_REGISTRY_EXTERNAL_PORT=32500
    ```

## **Installing k0rdent**

Manually install k0rdent using the following command:
```
helm  install  kcm  oci://ghcr.io/k0rdent/kcm/charts/kcm  --version  0.1.0  -n  k0rdent  --create-namespace
```

If your cluster already has cert-manager installed, you may want to disable k0rdent’s built-in cert-manager by appending:
```
--set cert-manager.enabled=false` 
```

If your cluster already has Flux CD installed,  **you must uninstall Flux CD or use a different cluster**.

Once the helm command has completed successfully, continue with the next step "Monitor the installation of k0rdent" in the "General" section.

## **Clean Up on a Non-Kind Cluster**

If you ever need to remove k0rdent resources and the associated Helm releases from a cluster that is **not** using Kind, you can follow this outline:

1. Tear down any managed clusters:
    ```
    make cleanup-clusters
    ```

2. Uninstall Helm Releases:
    ```
    for NAMESPACE in k0rdent projectsveltos mgmt; do  
        if kubectl get namespace $NAMESPACE >/dev/null 2>&1; then  
        # Uninstall any Helm releases in that namespace  
            for release in $(kubectl -n $NAMESPACE get helmreleases -o name | awk -F'/'  '{print $2}'); do  
                echo  "Uninstalling Helm release  $release  in namespace  $NAMESPACE" 
                helm uninstall $release -n $NAMESPACE || true  
            done  
        fi  
    done
    ```
    _Tip:_ Some CRDs might contain finalizers that can block namespace deletion.

3. Remove Finalizers from CRDs:

    The following example removes finalizers from the CRDs that might remain:
    ```
    CRD_DELETION_LIST=(
      helmcharts.source.toolkit.fluxcd.io
      helmreleases.helm.toolkit.fluxcd.io
      infrastructureproviders.operator.cluster.x-k8s.io
      bootstrapproviders.operator.cluster.x-k8s.io
      controlplaneproviders.operator.cluster.x-k8s.io
      provider.cluster.x-k8s.io
      coreproviders.operator.cluster.x-k8s.io
    )

    for NAMESPACE in k0rdent projectsveltos mgmt; do
        if kubectl get namespace $NAMESPACE >/dev/null 2>&1; then
            for crd in "${CRD_DELETION_LIST[@]}"; do
                if kubectl get "$crd" -n "$NAMESPACE" >/dev/null 2>&1; then
                    kubectl get "$crd" -n "$NAMESPACE" -o name | \
                        while read resource; do
                            kubectl patch "$resource" -n "$NAMESPACE" -p '{"metadata":{"finalizers":[]}}' --type=merge
                        done
                fi
            done
        fi
    done
    ```

4. Remove Any k0rdent Management Objects:

    If you have a management object named `kcm` (or similar):
    ```
    kubectl patch management.k0rdent kcm --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
    kubectl delete management.k0rdent kcm
    ```

5. Delete Remaining Namespaces:
    ```
    for NAMESPACE in k0rdent projectsveltos mgmt; do
        kubectl delete namespace "$NAMESPACE" --ignore-not-found
    done
    ```

6. Remove Sveltos / k0smotron CRDs:
    ```
    kubectl get crd -o jsonpath='{range .items[*]}{@.metadata.name}{"\n"}{end}' grep -E '(projectsveltos.io|k0smotron.io)' | while read -r crd; do
        echo "Deleting CRs for $crd"
        for resource in $(kubectl get "$crd" -A -o name 2>/dev/null); do
            kubectl patch "$resource" --type=merge -p '{"metadata":{"finalizers":[]}}'
            kubectl delete "$resource" --ignore-not-found
        done
        echo "Deleting CRD $crd"
        kubectl delete crd "$crd" --ignore-not-found
    done
    ```

    After these steps, the cluster should be cleared of k0rdent-related resources.
---
**Note:** These instructions are provided as a best-effort guide.