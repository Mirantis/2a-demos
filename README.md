# 2A Demo Repo

## What this is for

The intention of this 2A Demo repo is to have a place for demos and examples on how to leverage the Mirantis 2A Platform.

It includes scripts and implementation examples for basic and advanced usage for 2A.

All demos in here provide their own complete ClusterTemplates and ServiceTemplates and do not use the included 2A templates at all. This is done on one side to not be depending on 2A included templates and on the other side shows how custom and BYO (bring your own) templates can be used. Learn more about [BYO Templates in the 2A documentation](https://mirantis.github.io/project-2a-docs/template/byo-templates/).

## Setup

The Setup part for Demos is assumed to be created once before an actual demo is given.

Please make sure that docker is installed on your machine! It's required to run local kind cluster.

### General Setup

1. Create a 2A Management cluster with kind:
    ```shell
    make bootstrap-kind-cluster
    ```
    You could give it another name by specifying the `KIND_CLUSTER_NAME` environment variable.

2. Install 2A into kind cluster:
    ```shell
    make deploy-2a
    ```
    The Demos in this repo require at least 2A v0.0.5 or newer. You can change the version of 2A by specifying the `HMC_VERSION` environment variable.

3. Monitor the installation of 2A (you probably will need to install `jq` to execute this command):
    ```shell
    PATH=$PATH:./bin kubectl get management hmc -o json | jq -r '.status.components | to_entries[] | "\(.key): \(.value.success // .value.error)"'
    ```
   If the installation of 2a succeeded, the output should look as follows
   ```
   capi: true
   cluster-api-provider-aws: true
   cluster-api-provider-azure: true
   cluster-api-provider-vsphere: true
   hmc: true
   k0smotron: true
   projectsveltos: true
   ```

4. Install the Demo Helm Repo into 2A:
    ```shell
    make setup-helmrepo
    ```
    This step adds a [`HelmRepository` resource](https://fluxcd.io/flux/components/source/helmrepositories/) to the cluster that contains Helm charts for this demo.

### Infra Setup

As next you need to decide into which infrastructure you would like to install the Demo clusters. This Demo Repo has support for the following Infra Providers (more to follow in the future):

- AWS


#### AWS Setup

This assumes that you already have configured the required [AWS IAM Roles](https://mirantis.github.io/project-2a-docs/quick-start/aws/#configure-aws-iam) and have an [AWS account with the required permissions](https://mirantis.github.io/project-2a-docs/quick-start/aws/#step-1-create-aws-iam-user). If not follow the 2A documentation steps for them.

1. Export AWS Keys as environment variables:
    ```
    export AWS_ACCESS_KEY_ID="AKIAQIUDYGHDSJ3RZJC"
    export AWS_SECRET_ACCESS_KEY="hk8RAdjyfsiuhs7sG/kxLS+XS2xUHDUhfiuydZ4nSW"
    ````

2. Install Credentials into 2A:
    ```shell
    make setup-aws-creds
    ```

### Demo Cluster Setup

If your plan is to demo an upgrade (Demo 2) or anything related to ServiceTemplates (Demo 3 & 4) right after Demo 1, it is recommended to create a test cluster before the actual demo starts. The reason for this is that creation of a cluster takes around 10-15 mins and could cause a long waiting time during the demo. If you already have a second cluster you can show the creation of a cluster (Demo 1) and then use the existing cluster to show the other demos.


1. Install templates and create aws-test1 cluster
    ```shell
    make install-clustertemplate-demo-aws-standalone-cp-0.0.1
    make apply-aws-test1-0.0.1
    make watch-aws-test1
    ```

### Blue Namespace & Platform Engineer Credentials

If you plan to demo Demo 5 or above we need a secondary namespace (we call it blue in this demo) and credentials for a Platform Engineer that does only have access to the blue namespace and not cluster admin.

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
    KUBECONFIG="certs/platform-engineer1/kubeconfig.yaml" kubectl get ns blue
    ```

## Demo 1: Standalone Cluster Deployment

This demo shows how a simple standalone cluster from a custom ClusterTemplate can be created in the `hmc-system` namespace. It does not require any additional users in k8s or namespaces to be installed.

In the real world this would most probably be done by a Platform Team Lead that has admin access to the Management Cluster in order to create a test cluster from a new ClusterTemplate without the expectation for this cluster to exist for a long time.


1. Install ClusterTemplate in 2A
    ```shell
    make install-clustertemplate-demo-aws-standalone-cp-0.0.1
    ```
    This will install the custom ClusterTmplate and ClusterTemplateChain `demo-aws-standalone-cp-0.0.1` which exists in this Git Repo under `templates/cluster/demo-aws-standalone-cp-0.0.1` is hosted on the Github OCI registry at https://github.com/Mirantis/2a-demos.

    @TODO: add `kubectl -n hmc-system get clustertemplate`

    To make an even simpler Demo, this step could be done before the actual demo starts.

    As assumed by 2A all ClusterTemplates will be installed first into the `hmc-system` Namespace and can there be used directly to create a Cluster:

2. Install Test Clusters:
    ```shell
    make apply-aws-test1-0.0.1
    make apply-aws-test2-0.0.1
    ```
    This will create `ManagedCluster` with very simple defaults from the ClusterTemplate `demo-aws-standalone-cp-0.0.1`.
    The yaml for this can be found under `managedClusters/aws/1-0.0.1.yaml` and could be modified if needed.
    The Make command also shows the actual yaml that is created for an easier demo experience.


3. Monitor the deployment of the Cluster:
    ```shell
    make watch-aws-test2
    ```
    This will show the status and rollout of the cluster as seen by 2A.


4. Create Kubeconfig for Clusters:
    ```shell
    make get-kubeconfig-aws-test1
    make get-kubeconfig-aws-test2
    ````
    This will put a kubeconfig for a cluster admin under the folder `kubeconfigs`


5. Access Clusters through kubectl
    ```shell
    KUBECONFIG="kubeconfigs/hmc-system-aws-test1.kubeconfig" kubectl get pods -A
    ```

    ```shell
    KUBECONFIG="kubeconfigs/hmc-system-aws-test2.kubeconfig" kubectl get pods -A
    ```

## Demo 2: Single Standalone Cluster Upgrade

This demo shows how to upgrade an existing cluster through the cluster template system. This expects `Demo 1` to be completed or the `aws-test1` cluster already created during the Demo Setup.

This demo will upgrade the k8s cluster from `v1.31.1+k0s.1` (which is part of the `demo-aws-standalone-cp-0.0.1` template) to `v1.31.2+k0s.0` (which is part of `demo-aws-standalone-cp-0.0.2`)

1. Install ClusterTemplate Upgrade
    ```shell
    make install-clustertemplate-demo-aws-standalone-cp-0.0.2
    ```
    This will actually not only install a ClusterTemplate but also a ClusterTemplateChain. This ClusterTemplateChain will tell 2A that the `demo-aws-standalone-cp-0.0.2` is an upgrade from `demo-aws-standalone-cp-0.0.1`. You can see the source for it [here](templates/cluster/demo-aws-standalone-cp-0.0.2.yaml).


2. The fact that we have an upgrade available will be reported by 2A, and can be checked with:

    ```shell
    kubectl -n hmc-system get managedcluster.hmc.mirantis.com hmc-system-aws-test1 -o jsonpath='{.status.availableUpgrades}'
    ```

    @TODO: change command to load all clusters
    example output:
    ```
    [
      "demo-aws-standalone-cp-0.0.2"
    ]
    ```

3. Apply Upgrade of the cluster:
    ```shell
    make apply-aws-test1-0.0.2
    ```


4. Monitor the rollout of the upgrade
   ```shell
   KUBECONFIG="kubeconfigs/hmc-system-aws-test1.kubeconfig" kubectl get nodes --all-namespaces --watch
   ```

## Demo 3: Install ServiceTemplate into single Cluster

This demo shows how a ServiceTemplate can be installed in a Cluster.

In order to run this demo you need `Demo 1` completed, which created the `aws-test2` cluster.

1. Install ServiceTemplate in 2A:
    ```shell
    make install-servicetemplate-demo-ingress-nginx-4.11.0
    ```

2. Apply ServiceTemplate to cluster:
    ```shell
    make apply-aws-test2-0.0.1-ingress
    ```
    This applies the [0.0.1-ingress.yaml](managedClusters/aws/0.0.1-ingress.yaml) yaml template. For simplicity the yamls are a full `ManagedCluster` Object and not just a diff from the original cluster. The command output will show you a diff that explains that the only thing that actually has changed is the `serviceTemplate` key


3. Show that ingress-nginx is installed in the managed cluster:
    ```shell
    KUBECONFIG="kubeconfigs/hmc-system-aws-test2.kubeconfig" kubectl get pods -n ingress-nginx --watch
    ```


## Demo 4: Install ServiceTemplate into multiple Cluster

This Demo shows the capability of 2A to install a ServiceTemplate into multiple Clusters without the need to reference it in every cluster as we did in `Demo 3`.

While this demo can be shown even if you only have a single cluster, its obviously better to be demoed with two clusters. If you followed along the demo process you should have two clusters.

Be aware though that the cluster creation takes around 10-15mins, so depending on how fast you give the demo, the cluster creation might not be completed and the installation of services possible also delayed. You can totally follow this demo and the services will be installed after the clusters are ready.

1. Install Kyverno ServiceTemplate in 2A:
    ```shell
    make install-servicetemplate-demo-kyverno-3.2.6
    ```
    This will install a new servicetemplate which installs a standard installation of kyverno in a cluster. It has a clusterSelector configuration of the label `app.kubernetes.io/managed-by: Helm` which currently is the simplest way to match all clusters.

2. Apply MultiClusterService to cluster:
    ```shell
    make apply-multiclusterservice-global-kyverno
    ```

3. Show that kyverno is being installed in the two managed cluster:
    ```shell
    KUBECONFIG="kubeconfigs/hmc-system-aws-test1.kubeconfig" kubectl get pods -n kyverno
    ```

    ```shell
    KUBECONFIG="kubeconfigs/hmc-system-aws-test2.kubeconfig" kubectl get pods -n kyverno
    ```

    There might be a couple of seconds delay before that 2A and sveltos needs to start the installation of kyverno, give it at least 1 mins.

## Demo 5: Approve ClusterTemplate & InfraCredentials for separate Namespace

1. Approve the clustertemplate into the blue namespace
    ```shell
    make approve-clustertemplatechain-aws-standalone-cp-0.0.1
    ```

2. Approve the AWS credentials into the blue namspace
    ```shell
    make approve-credential-aws
    ```

3. Show that the platform engineer only can see the approved clustertemplate and no other ones:
    ```shell
    KUBECONFIG="certs/platform-engineer1/kubeconfig.yaml" kubectl get clustertemplates -n blue
    ```

## Demo 6: Use approved ClusterTemplate in separate Namespace

This demo is currently broken in HMC 0.0.5 until [#818](https://github.com/Mirantis/hmc/issues/818) is resolved.

1. Create Cluster in blue namespace (this will be ran as platform engineer)
    ```shell
    make apply-aws-dev1-0.0.1
    ```

2. Get Kubeconfig for `aws-dev1`
    ```shell
    make get-kubeconfig-aws-dev1
    ```

3. Access cluster
    ```shell
    KUBECONFIG="kubeconfigs/blue-aws-dev1.kubeconfig" kubectl get pods -A
    ```

## Demo 7: Test new clusterTemplate as 2A Admin, then approve them in separate Namespace

## Demo 8: Use newly approved Namespace in separate Namespace

## Demo 9: Approve ServiceTemplate in separate Namespace

## Demo 10: Use ServiceTemplate in separate Namespace



