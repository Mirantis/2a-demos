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
    ```
    make bootstrap-kind-cluster
    ```
    You could give it another name by specifying `KIND_CLUSTER_NAME` environment variable

2. Install 2A into kind cluster:
    ```
    make deploy-2a
    ```
    The Demos in this repo require at least 2A v0.0.5 or newer. You can change the version of the 2A by specifying the  

3. Monitor the installation of 2A (you probably will need to install `jq` to execute this command):
    ```
    PATH=$PATH:./bin kubectl get management hmc -o json | jq -r '.status.components | to_entries[] | "\(.key): \(.value.success // .value.error)"'
    ```
    @TODO: document what the output should look like

4. Install the Demo Helm Repo into 2A:
    ```
    make setup-helmrepo
    ```
    @TODO: describe what this does

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
    ```
    make setup-aws-creds
    ```

## Demo 1: Single Standalone Cluster Deployment

This demo show how a simple standalone cluster from a custom ClusterTemplate can be created in the `hmc-system` namespace. It does not require any additional users in k8s or namespaces to be installed.

In the real world this would most probably be done by a Platform Team Lead that has admin access to the Management Cluster in order to create a test cluster from a new ClusterTemplate without the expectation for this cluster to exist for a long time.


1. Install ClusterTemplate in 2A
    ```
    make install-clustertemplate-demo-aws-standalone-cp-0.0.1
    ```
    This will install the custom ClusterTmplate and ClusterTemplateChain `demo-aws-standalone-cp-0.0.1` which exists in this Git Repo under `templates/cluster/demo-aws-standalone-cp-0.0.1` is hosted on the Github OCI registry at https://github.com/Mirantis/2a-demos.

    @TODO: add `kubectl -n hmc-system get clustertemplate`

    To make an even simpler Demo, this step could be done before the actual demo starts.

    As assumed by 2A all ClusterTemplates will be installed first into the `hmc-system` Namespace and can there be used directly to create a Cluster:

2. Install Test Cluster:
    ```
    make apply-aws-test1-0.0.1
    ```
    This will create `ManagedCluster` with very simple defaults from the ClusterTemplate `demo-aws-standalone-cp-0.0.1`.
    The yaml for this can be found under `managedClusters/aws/1-0.0.1.yaml` and could be modified if needed.
    The Make command also shows the actual yaml that is created for an easier demo experience.


3. Monitor the deployment of the Cluster:
    ```
    make watch-aws-test1
    ```
    This will show the status and rollout of the cluster as seen by 2A.


4. Create Kubeconfig for Cluster:
    ```
    make get-kubeconfig-aws-test1
    ````
    This will put a kubeconfig for a cluster admin under the folder `kubeconfigs`


5. Access Cluster through kubectl
    ```
    KUBECONFIG="kubeconfigs/hmc-system-aws-test1.kubeconfig" kubectl get pods -A
    ```


## Demo 2: Single Standalone Cluster Upgrade

@TODO: explain that if this should be demoed a test cluster for the upgrade would probably need to be created already before this or we will need to wait 10-15min for the single cluster to be created

This demo shows how to upgrade an existing cluster through the cluster template system. This expects `Demo 1` to be completed.

This demo will upgrae the k8s cluster from `v1.31.1+k0s.1` (which is part of the `demo-aws-standalone-cp-0.0.1` template) to `v1.31.2+k0s.0` (which is part of `demo-aws-standalone-cp-0.0.2`)

1. Install ClusterTemplate Upgrade
    ```
    make install-clustertemplate-demo-aws-standalone-cp-0.0.2
    ```
    This will actually not only install a ClusterTemplate but also a ClusterTemplateChain. This ClusterTemplateChain will tell 2A that the `demo-aws-standalone-cp-0.0.2` is an upgrade from `demo-aws-standalone-cp-0.0.1`. You can see the source for it [here](templates/cluster/demo-aws-standalone-cp-0.0.2.yaml).


2. The fact that we have an upgrade available will be reported by 2A, and can be checked with:

    ```
    kubectl -n hmc-system get managedcluster.hmc.mirantis.com hmc-system-aws-test1  -o json | jq -r '.status.availableUpgrades'
    ```

    @TODO: change command to load all clusters
    example output:
    ```
    [
      "demo-aws-standalone-cp-0.0.2"
    ]
    ```

3. Apply Upgrade of the cluster:
    ```
    make apply-aws-test1-0.0.2
    ```


4. Monitor the rollout of the upgrade @TODO: find a way to monitor the rollout of the upgrade


## Demo 3: Install ServiceTemplate into single Cluster

This demo shows how a ServiceTemplate can be installed in a Cluster.

In order to run this demo you need `Demo 1` and/or `Demo 1 & 2` already completed.

1. Install ServiceTemplate in 2A:
    ```
    make install-servicetemplate-demo-ingress-nginx-4.11.0
    ```


2. Apply ServiceTemplate to cluster:
    ```
    make apply-aws-test1-0.0.2-ingress
    ```
    Or if you just have completed Demo 1
    ```
    make apply-aws-test1-0.0.1-ingress
    ```
    This applies either the [0.0.1-ingress.yaml](managedClusters/aws/0.0.1-ingress.yaml) or [0.0.2-ingress.yaml](managedClusters/aws/0.0.2-ingress.yaml) yaml template. For simplicity the yamls are a full `ManagedCluster` Object and not just a diff from the original cluster. The command output will show you a diff that explains that the only thing that actually has chnaged is the `serviceTemplate` key


3. Show that ingress-nginx is installed in the managed cluster:
    ```
    KUBECONFIG="kubeconfigs/hmc-system-aws-test1.kubeconfig" kubectl get pods -n ingress-nginx
    ```


## Demo 4: Install ServiceTemplate into multiple Cluster

This Demo shows the capability of 2A to install a ServiceTemplate into multiple Clusters without the need to reference it in every cluster as we did in `Demo 3`. Most probably we only have a single cluster at this point which defeats the idea to show something in multiple Clusters. For this purpose there is the possiblity to create a second test cluster through `make apply-aws-test2-0.0.1`. Be aware though that the cluster creation takes around 10-15mins, so if you continue right away with the MultiClusterService you might not be able to show both clusters right away.


## Demo 5: Approve ClusterTemplate & InfraCredentials for separate Namespace


## Demo 6: Use approved ClusterTemplate in separate Namespace

## Demo 7: Test new clusterTemplate as 2A Admin, then approve them in separate Namespace

## Demo 8: Use newwly approved Namespace in separate Namespace

## Demo 9: Approve ServiceTemplate in separate Namespace

## Demo 10: Use ServiceTemplate in separate Namespace



