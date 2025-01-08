# k0rdent Demo Repo

## What this is for

The intention of this k0rdent Demo repo is to have a place for demos and examples on how to leverage the Mirantis k0rdent Project.

It includes scripts and implementation examples for basic and advanced usage for k0rdent.

All demos in here provide their own complete ClusterTemplates and ServiceTemplates and do not use the included k0rdent templates at all. This is done on one side to not be depending on k0rdent included templates and on the other side shows how custom and BYO (bring your own) templates can be used. Learn more about [BYO Templates in the k0rdent documentation](https://k0rdent.github.io/docs/template/byo-templates/).

## Setup

The Setup part for Demos is assumed to be created once before an actual demo is given.

To get the full list of commands run `make help`.

### General Setup

> Expected completion time ~10 min

1. Create a k0rdent Management cluster with kind:
    ```shell
    make bootstrap-kind-cluster
    ```
    You could give it another name by specifying the `KIND_CLUSTER_NAME` environment variable. 
    
    For detailed explanation, please refer to the [documentation](./documentation/1-general-setup-bootstrap-kind-cluster.md).

2. Install k0rdent into kind cluster:
    ```shell
    make deploy-k0rdent
    ```
    The Demos in this repo require at least k0rdent v0.0.6 or newer. You can change the version by specifying the `KCM_VERSION` environment variable. List of releases can be found [here](https://github.com/K0rdent/kcm/releases).

3. Monitor the installation of k0rdent:
    ```shell
    make watch-k0rdent-deployment
    ```
    In this command we track the `Management` object that is created by k0rdent. Don't worry if you get message that the object is not found, it can take some time.
    Wait until the output of the command be as follows to make sure that k0rdent project is fully installed:
    ```
    capi: true
    cluster-api-provider-aws: true
    cluster-api-provider-azure: true
    cluster-api-provider-vsphere: true
    hmc: true
    k0smotron: true
    projectsveltos: true
    ```
    For detailed explanation, please refer to the [documentation](./documentation/2-general-setup-deploy-k0rdent.md).

4. Install the Demo Helm Repo into k0rdent:
    ```shell
    make setup-helmrepo
    ```
    This step deploys simple local OCI Helm registry and adds a [`HelmRepository` resource](https://fluxcd.io/flux/components/source/helmrepositories/) to the cluster that contains Helm charts for this demo.

    For detailed explanation, please refer to the [documentation](./documentation/3-general-setup-helmrepo-setup.md).

5. Push Helm charts with custom Cluster and Service Templates
    ```
    make push-helm-charts
    ```


### Infra Setup

> Expected completion time ~2 min

As next you need to decide into which infrastructure you would like to install the Demo clusters. This Demo Repo has support for the following Infra Providers (more to follow in the future):

- AWS
- Azure

#### AWS Setup

This assumes that you already have configured the required [AWS IAM Roles](https://k0rdent.github.io/docs/quick-start/aws/#configure-aws-iam) and have an [AWS account with the required permissions](https://k0rdent.github.io/docs/quick-start/aws/#step-1-create-aws-iam-user). If not follow the k0rdent documentation steps for them.

1. Export AWS Keys as environment variables:
    ```shell
    export AWS_ACCESS_KEY_ID="AWS Access Key ID"
    export AWS_SECRET_ACCESS_KEY="AWS Secret Access Key"
    ````
2. By default, it will provision all resources in the `us-west-2` AWS region. If you want to change this, export `AWS_REGION` environment variable:
    ```shell
    export AWS_REGION="us-east-1"
    ```

3. Install Credentials into k0rdent:
    ```shell
    make setup-aws-creds
    ```

4. Check that credentials are ready to use
    ```shell
    make get-creds-aws
    ```
    The output should be similar to:
    ```
    NAME                        READY   DESCRIPTION
    aws-cluster-identity-cred   true    Basic AWS credentials
    ```

#### Azure Setup

**Currently demos don't have Azure cluster deployments, so you can skip this section**

This assumes that you already have configured the required [Azure providers](https://k0rdent.github.io/docs/quick-start/azure/#register-resource-providers) and created a [Azure Service Principal](https://k0rdent.github.io/docs/quick-start/azure/#step-2-create-a-service-principal-sp).

1. Export Azure Service Principal keys as environment variables:
    ```
    export AZURE_SP_PASSWORD=<Service Principal password>
    export AZURE_SP_APP_ID=<Service Principal App ID>
    export AZURE_SP_TENANT_ID=<Service Principal Tenant ID>
    export AZURE_SUBSCRIPTION_ID=<Azure's subscription ID>
    ```

2. Install Credentials into k0rdent:
    ```
    make setup-azure-creds
    ```

3. Check that credentials are ready to use
    ```shell
    make get-creds-azure
    ```
    The output should be similar to:
    ```
    NAME                          READY   DESCRIPTION
    azure-cluster-identity-cred   true    Azure credentials
    ```

### Demo Cluster Setup

**Skip this step if you just want to run demos for your own**

If your plan is to demo an upgrade (Demo 2) or anything related to ServiceTemplates (Demo 3 & 4) right after Demo 1, it is recommended to create a test cluster before the actual demo starts. The reason for this is that creation of a cluster takes around 10-15 mins and could cause a long waiting time during the demo. If you already have a second cluster you can show the creation of a cluster (Demo 1) and then use the existing cluster to show the other demos.


1. Install templates and create aws-test1 cluster
    ```shell
    make apply-clustertemplate-demo-aws-standalone-cp-0.0.1
    make apply-cluster-deployment-aws-test1-0.0.1
    make watch-aws-test1
    ```
2. Wait when the cluster deployment be in Ready state:
    ```
    NAME                   READY   STATUS
    k0rdent-aws-test1      True    ClusterDeployment is ready
    ```

### Blue Namespace & Platform Engineer Credentials

If you plan to run the [`Demo 5`](#demo-5-approve-clustertemplate--infracredentials-for-separate-namespace) or above we need a secondary namespace (we call it `blue` in this demo) and credentials for a Platform Engineer that does only have access to the blue namespace and not cluster admin.

1. Create target namespace blue and required rolebindings
    ```shell
    make create-target-namespace-rolebindings
    ```

2. Generate Kubeconfig for platform engineer
    ```shell
    make clean-certs
    make generate-platform-engineer1-kubeconfig
    ```

3. Test Kubeconfig
    ```shell
    KUBECONFIG="certs/platform-engineer1/kubeconfig.yaml" PATH=$PATH:./bin kubectl get ns blue
    ```

## Demo 1: Standalone Cluster Deployment

> Expected completion time ~10-15 min

This demo shows how a simple standalone cluster from a custom ClusterTemplate can be created in the `k0rdent` namespace. It does not require any additional users in k8s or namespaces to be installed.

In the real world this would most probably be done by a Platform Team Lead that has admin access to the Management Cluster in order to create a test cluster from a new ClusterTemplate without the expectation for this cluster to exist for a long time.


1. Install ClusterTemplate in k0rdent
    ```shell
    make apply-clustertemplate-demo-aws-standalone-cp-0.0.1
    ```
    This will install the custom [ClusterTemplate and ClusterTemplateChain](./templates/cluster/demo-aws-standalone-cp-0.0.1.yaml) `demo-aws-standalone-cp-0.0.1`. ClusterTemplate refers to the [Helm chart](./templates/cluster/demo-aws-standalone-cp-0.0.1/) `demo-aws-standalone-cp` of version `0.0.1` which was published to the local Helm repository on the Infra Setup steps.

    You can find this new ClusterTemplate in the list of template with the command:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get clustertemplates
    ```

    Example of the output:
    ```
    NAME                           VALID
    ...
    demo-aws-standalone-cp-0.0.1   true
    ...
    ```

    > Hint: To make an even simpler Demo, this step could be done before the actual demo starts.

    As assumed by k0rdent all ClusterTemplates will be installed first into the `k0rdent` Namespace and can there be used directly to create a new cluster deployments.

2. Install Test Clusters:
    ```shell
    make apply-cluster-deployment-aws-test1-0.0.1
    make apply-cluster-deployment-aws-test2-0.0.1
    ```
    This will create 2 objects of type `ClusterDeployment` with very simple defaults from the ClusterTemplate `demo-aws-standalone-cp-0.0.1`.
    The yaml for this can be found under [`clusterDeployments/aws/0.0.1.yaml`](./clusterDeployments/aws/0.0.1.yaml) and could be modified if needed.
    The Make command also shows the actual yaml that is created for an easier demo experience.

3. Monitor the deployment of each cluster and wait when both be in Ready state:
    For the first cluster:
    ```shell
    make watch-aws-test1
    ```

    Example of the output of fully deployed first cluster:
    ```
    NAME                READY   STATUS
    k0rdent-aws-test1   True    ClusterDeployment is ready
    ```

    For the seocnd cluster:
    ```shell
    make watch-aws-test2
    ```
    
    Example of the output of fully deployed second cluster:
    ```
    NAME                READY   STATUS
    k0rdent-aws-test2   True    ClusterDeployment is ready
    ```

4. Create Kubeconfig for Clusters:
    ```shell
    make get-kubeconfig-aws-test1
    make get-kubeconfig-aws-test2
    ```
    This will put kubeconfigs for a cluster admin under the folder `kubeconfigs` for both created clusters


5. Access Clusters through kubectl
    ```shell
    KUBECONFIG="kubeconfigs/k0rdent-aws-test1.kubeconfig" PATH=$PATH:./bin kubectl get node
    ```

    Example output:
    ```
    NAME                               STATUS   ROLES           AGE   VERSION
    k0rdent-aws-test1-cp-0             Ready    control-plane   19m   v1.31.2+k0s
    k0rdent-aws-test1-md-j87z9-fljb4   Ready    <none>          17m   v1.31.2+k0s
    k0rdent-aws-test1-md-j87z9-r85gs   Ready    <none>          17m   v1.31.2+k0s
    ```

    ```shell
    KUBECONFIG="kubeconfigs/k0rdent-aws-test2.kubeconfig" PATH=$PATH:./bin kubectl get node
    ```

    Example output:
    ```
    NAME                               STATUS   ROLES           AGE   VERSION
    k0rdent-aws-test2-cp-0             Ready    control-plane   19m   v1.31.2+k0s
    k0rdent-aws-test2-md-j87z9-fljb4   Ready    <none>          17m   v1.31.2+k0s
    k0rdent-aws-test2-md-j87z9-r85gs   Ready    <none>          17m   v1.31.2+k0s
    ```

## Demo 2: Single Standalone Cluster Upgrade

> Expected completion time ~10 min

This demo shows how to upgrade an existing cluster through the cluster template system. This expects [Demo 1](#demo-1-standalone-cluster-deployment) to be completed or the `aws-test1` cluster already created during the [Demo Setup](#demo-cluster-setup).

This demo will upgrade the k8s cluster from `v1.31.1+k0s.1` (which is part of the `demo-aws-standalone-cp-0.0.1` template) to `v1.31.2+k0s.0` (which is part of `demo-aws-standalone-cp-0.0.2`)

1. Install ClusterTemplate Upgrade
    ```shell
    make apply-clustertemplate-demo-aws-standalone-cp-0.0.2
    ```
    This will actually not only install a [ClusterTemplate but also a ClusterTemplateChain](./templates/cluster/demo-aws-standalone-cp-0.0.2.yaml). This ClusterTemplateChain will tell k0rdent that the `demo-aws-standalone-cp-0.0.2` is an upgrade from `demo-aws-standalone-cp-0.0.1`.

    You can find this new ClusterTemplate in the list of template with the command:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get clustertemplates
    ```

    Example of the output:
    ```
    NAME                           VALID
    ...
    demo-aws-standalone-cp-0.0.1   true
    demo-aws-standalone-cp-0.0.2   true <-- this is the new template
    ...
    ```

2. The fact that we have an upgrade available will be reported by k0rdent. You can find all available upgrades for all cluster deployments by executing this command:

    ```shell
    make get-avaliable-upgrades
    ```

    Example output:
    ```
    Cluster k0rdent-aws-test1 available upgrades: 
      - demo-aws-standalone-cp-0.0.2

    Cluster k0rdent-aws-test2 available upgrades: 
      - demo-aws-standalone-cp-0.0.2
    ```

3. Apply Upgrade of the cluster:
    ```shell
    make apply-cluster-deployment-aws-test1-0.0.2
    ```

4. Monitor the rollout of the upgrade

    You can watch how new machines are created and old machines are deleted:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get machines -w
    ```

    You can see how for cluster `test1` the k0s control plane node version is upgraded to the new one, then one by one new worker nodes should be provisioned and put into `Running` state, and old nodes should be removed.

    > Hint: control plane nodes for k0s clusters are being upgraded in place (check the version field) without provisioning new machines.

    And how in the created cluster old nodes are drained and new nodes are attached:
    ```shell
    KUBECONFIG="kubeconfigs/k0rdent-aws-test1.kubeconfig" PATH=$PATH:./bin kubectl get node -w
    ```

## Demo 3: Install ServiceTemplate into single Cluster

> Expected completion time ~5 min

This demo shows how a ServiceTemplate can be installed in a Cluster.

In order to run this demo you need [`Demo 1`](#demo-1-standalone-cluster-deployment) completed, which created the `test2` cluster.

1. Install ServiceTemplate in k0rdent:
    ```shell
    make apply-servicetemplate-demo-ingress-nginx-4.11.0
    ```

    This installs the custom [ServiceTemplate and ServiceTemplateChain](./templates/service/demo-ingress-nginx-4.11.0.yaml) `demo-ingress-nginx-4.11.0`. ServiceTemplate refers to the [Helm chart](./templates/service/demo-ingress-nginx-4.11.0/) `demo-ingress-nginx` of version `4.11.0` which was published to the local Helm repository on the Infra Setup steps.

    You can find this new ServiceTemplate in the list of template with the command:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get servicetemplates
    ```

    Example of the output:
    ```
    NAME                        VALID
    ...
    demo-ingress-nginx-4.11.0   true <-- this is the installed template
    ...
    ```

2. Apply ServiceTemplate to cluster:
    ```shell
    make apply-cluster-deployment-aws-test2-0.0.1-ingress
    ```
    This applies the [0.0.1-ingress.yaml](clusterDeployments/aws/0.0.1-ingress.yaml) yaml template. For simplicity the yamls are a full `ClusterDeployment` Object and not just a diff from the original cluster. The command output will show you a diff that explains that the only thing that actually has changed is the `serviceTemplate` key


3. Monitor how the ingress-nginx is installed in `test2` cluster:
    ```shell
    watch KUBECONFIG="kubeconfigs/k0rdent-aws-test2.kubeconfig" PATH=$PATH:./bin kubectl get pods -n ingress-nginx
    ```

    The final state should be similar to:
    ```
    NAME                                        READY   STATUS    RESTARTS   AGE
    ingress-nginx-controller-86bd747cf9-ds56s   1/1     Running   0          34s
    ```

    You can also check the services status of the `ClusterDeployment` of object in management cluster:

    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get clusterdeployment.hmc.mirantis.com k0rdent-aws-test2 -o yaml
    ```

    The output under the `status.services` should contain information about successfully deployed ingress nginx service:

    ```
    ...
    status:
      ...
      services:
      - clusterName: k0rdent-aws-test2
        clusterNamespace: k0rdent
        conditions:
        ...
        - lastTransitionTime: "2024-12-19T17:24:35Z"
          message: Release ingress-nginx/ingress-nginx
          reason: Managing
          status: "True"
          type: ingress-nginx.ingress-nginx/SveltosHelmReleaseReady
    ```


## Demo 4: Install ServiceTemplate into multiple Cluster

This Demo shows the capability of k0rdent to install a ServiceTemplate into multiple Clusters without the need to reference it in every cluster as we did in `Demo 3`.

While this demo can be shown even if you only have a single cluster, its obviously better to be demoed with two clusters. If you followed along the demo process you should have two clusters.

Be aware though that the cluster creation takes around 10-15mins, so depending on how fast you give the demo, the cluster creation might not be completed and the installation of services possible also delayed. You can totally follow this demo and the services will be installed after the clusters are ready.

1. Install Kyverno ServiceTemplate in k0rdent:
    ```shell
    make apply-servicetemplate-demo-kyverno-3.2.6
    ```
    This will install a new [ServiceTemplate](./templates/service/demo-kyverno-3.2.6.yaml) which installs a standard installation of kyverno in a cluster.

    You can find this new ServiceTemplate in the list of template with the command:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get servicetemplates
    ```

    Example of the output:
    ```
    NAME                        VALID
    ...
    demo-ingress-nginx-4.11.0   true 
    demo-kyverno-3.2.6          true <-- this is the installed template
    ...
    ```

2. Apply MultiClusterService to cluster:
    ```shell
    make apply-multiclusterservice-global-kyverno
    ```

    This will install a `hmc.mirantis.com/v1alpha1/MultiClusterService` cluster-wide object to the management cluster. It has a clusterSelector configuration of the label `app.kubernetes.io/managed-by: Helm` which selects all `cluster.x-k8s.io/v1beta1/Cluster` objects with this label. Please, don't confuse `hmc.mirantis.com/v1alpha1/ClusterDeployment` and `cluster.x-k8s.io/v1beta1/Cluster` types. First one - is the type of Project k0rdent objects, we deploy them to the management cluster and then, kcm operator creates various objects, including CAPI `cluster.x-k8s.io/v1beta1/Cluster`. `hmc.mirantis.com/v1alpha1/MultiClusterService` relies on `cluster.x-k8s.io/v1beta1/Cluster` labels. Currently, it's not possible to specify them in the `ClusterDeployment` object configuration, there is an [issue](https://github.com/Mirantis/hmc/issues/801) on GitHub. But, to demonostrate the possibility of deploying service to multiple clusters without specifiying in each `ClusterDeployment` object, we will use in this demo the `app.kubernetes.io/managed-by: Helm` label, which is automatically set to all `cluster.x-k8s.io/v1beta1/Cluster` objects by k0rdent.

3. Monitor how the kyverno service is being installed in both clusters that we deployed previously:
    ```shell
    watch KUBECONFIG="kubeconfigs/k0rdent-aws-test1.kubeconfig" kubectl get pods -n kyverno
    ```

    ```shell
    watch KUBECONFIG="kubeconfigs/k0rdent-aws-test2.kubeconfig" kubectl get pods -n kyverno
    ```

    There might be a couple of seconds delay before that k0rdent and sveltos needs to start the installation of kyverno, give it at least 1 mins.

    The final state for each cluster should be similar to:
    ```
    NAME                                             READY   STATUS    RESTARTS   AGE
    kyverno-admission-controller-96c5d48b4-rqpdz     1/1     Running   0          47s
    kyverno-background-controller-65f9fd5859-qfwqc   1/1     Running   0          47s
    kyverno-cleanup-controller-848b4c579d-fc8s4      1/1     Running   0          47s
    kyverno-reports-controller-6f59fb8cd6-9j4f7      1/1     Running   0          47s
    ```    

4. You can also check the deployment status for all clusters in the `MultiClusterService` object:
    ```shell
    PATH=$PATH:./bin kubectl get multiclusterservice global-kyverno -o yaml
    ```

    In the output you can find information about clusters where the service is deployed:
    ```
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: MultiClusterService
    ...
    status:
      ...
      services:
        - clusterName: k0rdent-aws-test1
          clusterNamespace: k0rdent
          conditions:
          - lastTransitionTime: "2025-01-03T14:12:33Z"
            message: ""
            reason: Provisioned
            status: "True"
            type: Helm
          - lastTransitionTime: "2025-01-03T14:12:33Z"
            message: Release kyverno/kyverno
            reason: Managing
            status: "True"
            type: kyverno.kyverno/SveltosHelmReleaseReady
        - clusterName: k0rdent-aws-test2
          clusterNamespace: k0rdent
          conditions:
          - lastTransitionTime: "2025-01-03T14:12:33Z"
            message: ""
            reason: Provisioned
            status: "True"
            type: Helm
          - lastTransitionTime: "2025-01-03T14:12:33Z"
            message: Release kyverno/kyverno
            reason: Managing
            status: "True"
            type: kyverno.kyverno/SveltosHelmReleaseReady
    ```

## Demo 5: Approve ClusterTemplate & InfraCredentials for separate Namespace

1. Approve the clustertemplate into the blue namespace
    ```shell
    make approve-clustertemplatechain-aws-standalone-cp-0.0.1
    ```

    Check the status of the `hmc` AccessManagement object:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get AccessManagement hmc -o yaml
    ```

    In the status section you can find information about the clustertemplate that was approved to the target `blue` namespace:
    ```
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: AccessManagement
    ...
    status:
      ...
      current:
      - clusterTemplateChains:
        - demo-aws-standalone-cp-0.0.1
        targetNamespaces:
          list:
          - blue
    ```

2. Approve the AWS credentials into the blue namspace
    ```shell
    make approve-credential-aws
    ```

    Check the status of the `hmc` AccessManagement object:
    ```shell
    PATH=$PATH:./bin kubectl -n k0rdent get AccessManagement hmc -o yaml
    ```

    In the status section you can find information about the clustertemplate that was approved to the target `blue` namespace:
    ```
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: AccessManagement
    ...
    status:
      ...
      current:
      - credentials:
        - aws-cluster-identity-cred
        targetNamespaces:
          list:
          - blue
    ```

3. Show that the platform engineer only can see approved clustertemplate and credentials and no other ones:
    ```shell
    KUBECONFIG="certs/platform-engineer1/kubeconfig.yaml" PATH=$PATH:./bin kubectl get credentials,clustertemplates -n blue
    ```

    Output:
    ```
    NAME                                                    READY   DESCRIPTION
    credential.hmc.mirantis.com/aws-cluster-identity-cred   true    AWS credentials

    NAME                                                            VALID
    clustertemplate.hmc.mirantis.com/demo-aws-standalone-cp-0.0.1   true
    ```

## Demo 6: Use approved ClusterTemplate in separate Namespace

1. Create Cluster in blue namespace (this will be ran as platform engineer)
    ```shell
    make apply-cluster-deployment-aws-dev1-0.0.1
    ```
    This will create the object of type `ClusterDeployment`, same as in [`Demo 1`](#demo-1-standalone-cluster-deployment) but in the `blue` namespace and using the approved and delivered `ClusterTemplate` to that namespace.

3. Monitor the deployment of the cluster and wait when both be in Ready state:
    For the first cluster:
    ```shell
    make watch-aws-dev1
    ```

    Example of the output of fully deployed first cluster:
    ```
    NAME            READY   STATUS
    blue-aws-dev1   True    ClusterDeployment is ready
    ```


2. Get Kubeconfig for `aws-dev1`
    ```shell
    make get-kubeconfig-aws-dev1
    ```

3. Access cluster
    ```shell
    KUBECONFIG="kubeconfigs/blue-aws-dev1.kubeconfig" kubectl get node
    ```

    Example output:
    ```
    NAME                           STATUS   ROLES           AGE    VERSION
    blue-aws-dev1-cp-0             Ready    control-plane   10m    v1.31.2+k0s
    blue-aws-dev1-md-kxgdb-bgmhw   Ready    <none>          2m4s   v1.31.2+k0s
    blue-aws-dev1-md-kxgdb-jkcpc   Ready    <none>          2m5s   v1.31.2+k0s
    ```

## Demo 7: Test new clusterTemplate as k0rdent Admin, then approve them in separate Namespace

## Demo 8: Use newly approved Namespace in separate Namespace

## Demo 9: Approve ServiceTemplate in separate Namespace

## Demo 10: Use ServiceTemplate in separate Namespace

## Cleaning up

As running the whole k0rdent setup can be quite taxing on your hardware, run the following command to clean up everything (both the public cloud resources mentioned above but also all local containers):
```shell
  make cleanup
```
