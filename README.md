# Setup Jenkins-X + hybris on a GKE cluster

This document provides a how-to setup your own hybris cluster inside the Google
Kubernetes Engine (GKE)

## Prerequisites

- You have a Google Cloud Engine Account
- You have a Google Cloud Project
- Cloud SQL Administration API is [activated](gsql-quickstart) for your project
- Compute Engine is activated for your project
  (This happens as soon as you open the Compute Engine Dashboard)
- Kubernetes Engine API is activated for your project (Open the Kubernetes Engine Dashboard to trigger the init)
- `glcoud` command line tools are [installed](gcloud-install)
  and [initialized](glcoud-init)
- Jenkins-X is [installed](jx-install)

This guide assumes you have set your region and zone preferences in your gcloud
configuration profile!


## Initial setup

We will start with setting up the Kubernetes Cluster and by creating a Cloud SQL instance.

Feel free to inspect and change any of the configuration files as you see fit.
The values provided are good enough for a demo, but not for a production system.

### Create and configure Kubernetes Cluster

1. Create cluster

        gcloud container clusters create \
        hybris \
        --machine-type n1-highmem-2 \
        --num-nodes 4 \
        --enable-autoscaling \
        --max-nodes 4 \
        --min-nodes 0 \
        --cluster-version 1.10.4-gke.0
    
    Please use the latest Kubernetes version available ( `--cluster-version`).
    You can see all supported versions via `gcloud container get-server-config`

    I use only 4 nodes in this example because if you use the free trial of Google
    Cloud, you are limited to 8 cores per zone. Ideally, you would have more powerful
    nodes for the hybris servers.
    
    Or, you could try setting up a [multi-zone/region cluster](gke-multi) to 
    circumvent the limits per region, but I haven't tried that
    

1. Setup `kubectl` context configuration for the newly created cluster

        gcloud container clusters get-credentials hybris

### Install additional services via helm

To provide a fully featured demo, we need some additional services inside the newly created cluster.
[Helm](helm) makes installing those a breeze.

1. Setup and install helm for the new cluster with a [dedicated service account](helm-rbac)

        kubectl apply -f initial-setup/tiller-service-account.yaml
        helm init --service-account tiller

1. Setup and install [NFS provisioner](nfs-provisioner)

        helm install \
        stable/nfs-server-provisioner \
        --name nfs-provisioner \
        --namespace kube-system \
        -f initial-setup/nfs-provisioner.yaml

    You may ask yourself: *Why do we need the nfs server provisioner? GKE already provides persistent volumes.*

    The answer is the hybris media folder. The default persistent volume
    provisioner of GKE only supports the Access Modes `ReadWriteOnce` and `ReadOnlyMany`.
    But, the hybris media folder can be accessed by all nodes in read/write mode.
    (e.g. multiple backend nodes generate media files, or the user can upload files
    that are saved as medias in the system)

    The NFS server provisioner supports `ReadWriteMany`, it basically wraps an NFS server 
    around a GKE disk.

1. Setup and install [NGINX ingress controller](nginx)

        helm install \
        stable/nginx-ingress \
        --name nginx-ingress \
        --namespace kube-system  \
        --set rbac.create=true

    We use nginx ingresses to expose the various endpoints (of Jenkins-X and the hybris environments)
    to the internet.
    The nginx ingress allows quite some customization and also TLS termination.

1. Setup and install [cert-manager](cert-manager)

        helm install \
        stable/cert-manager \
        --name cert-manager \
        --namespace kube-system

        kubectl apply -f initial-setup/acme-staging-issuer.yaml
        kubectl apply -f initial-setup/acme-prod-issuer.yaml

    **Important**: Change the personal information in `initial-setup/acme-staging-issuer.yaml`
    and `initial-setup/acme-prod-issuer.yaml`

    We use the cert-manager to automatically generate TLS certificates
    via ACME (Let's Encrypt) for the ingresses generated via the nginx ingress controller.

### Bootstrap Jenkins-X

Here most of the magic happens. [`jx install`](https://jenkins-x.io/getting-started/install-on-cluster/) bootstraps a complete development
pipeline and tooling inside your Kubernetes cluster.

        jx install \
        --skip-tiller=true \
        --helm-client-only=true \
        --provider=gke \
        --ingress-deployment=nginx-ingress-controller \
        --ingress-service=nginx-ingress-controller \
        --no-default-environments=true

Since we already installed helm and the latest version of the nginx ingress controller,
we provide some flags to `jx install` to skip the installation for those components.

Also, we don't want to have any default environments, as we have our own setup for those.

The install process will prompt you for some information, use the default values
except for the GitHub credentials.

After it is done, run `jx console` to open the newly installed Jenkins
and to verify the setup was successful

Since the demo also uses SonarQube in its build, let's install it inside the
the `jx` namespace, where all the build tools live:

    helm install stable/sonarqube \
    --name sonar \
    --namespace jx \
    --set service.type=ClusterIP

#### Manual Configuration

Unfortunately, there are some manual steps necessary to prepare Jenkins-X for the
hybris build, as the build is quite heavy and special...

1. Increase memory for gradle build pod
    
    1. Go to http://jenkins.jx.XXX.XXX.XXX.XXX.nip.io/configure
    1. Log in as admin, if necessary
    1. Search for the "Kubernetes Pod Template" named `gradle`
    1. Go to "Containers", klick "Advanced"
    1. Change Container Limits

        |Property|Value|
        |-|-|
        |Request CPU|1|
        |Request Memory|2Gi|
        |Limit CPU|2|
        |Limit Memory|3Gi|

    1. Change EnvVar `_JAVA_OPTIONS` to

            -XX:MaxRAMFraction=2 -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -XX:+UseParallelGC -XX:MinHeapFreeRatio=5 -XX:MaxHeapFreeRatio=10 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90

1. Persistent Volume for gradle build cache **[OPTIONAL, EXPERIMENTAL]**

    This steps are optional and may break gradle!

    1. Create Persistent Volume

            kubectl apply -f initial-setup/gradle-cache-pvc.yaml

    1. Go to http://jenkins.jx.XXX.XXX.XXX.XXX.nip.io/configure
    1. Log in as admin, if necessary
    1. Search for the "Kubernetes Pod Template" named `gradle`
    1. Add a new "Volume" via "Add Volume" -> "Persistent Volume Claim"

        |Property|Value|
        |-|-|
        |Claim Name|gradle-cache|
        |Read Only|false|
        |Mount Path|/root/.gradle/caches|

    Gradle caches maven artifacts in this folder. Providing a Persistent Volume
    for it helps reduce build times, especially for hybris, as the platform zip is
    huge and takes a while to download

1. Add secrets

    1. Go to http://jenkins.jx.XXX.XXX.XXX.XXX.nip.io/credentials/store/system/domain/_/
    1. Login as admin, if necessary
    1. Create two "Username with password" secrets

        |Username|Password|ID|Description
        |-|-|-|-|
        |admin|\<jenkins-x-admin-password\>|nexus|Login for Maven Nexus|
        |\<s-user-id\>|\<s-user-password\>|support-portal-user|S-User to download hybris platform zips from SAP Support Portal|

### Create a Cloud SQL (MySQL) instance

The demo uses one CloudSQL instance to host all hybris databases. Here are the 
steps to set this up.

1. Create the instance

        gcloud sql instances create \
        hybris-database-test \
        --tier=db-n1-standard-4 \
        --database-version MYSQL_5_7 \
        --region=europe-west3 \
        --gce-zone=europe-west3-a

    **Important** Adapt the `region` and `gce-zone` parameters to match the region and zone of
    your GKE cluster (to minimize latency)

1. Set root password

        gcloud sql users set-password root % --instance hybris-database-test --password your-root-password

1. Create a service account to connect from GKE

    https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine

    Follow the steps of "2. Create a service account" until you have downloaded the service account key as json file
    The role `Cloud SQL Client` is sufficient.

    I will reference the generate key file as `PROXY_KEY_FILE_PATH` in the rest
    of the documentation.

This concludes the initial setup.
You now have:

- A Kubernetes cluster with additional tools for a fully featured hybris environment
- A running, empty Jenkins-X installation inside your cluster
- A Cloud SQL (MySQL) server that will host the databases for your hybris installations

## Import Demo Projects

Now the fun part! Let's import the demo projects.

### Hybris Version Download

https://github.com/hybris-jenkins-x-demo/hybris-version-download

1. Fork the project into your own GitHub account or organization.
1. Clone it into a folder on your machine
1. `cd` into the local clone
1. run `jx import --no-draft --no-jenkinsfile` and follow the prompts to add the project to your Jenkins-X installation

After the import is finished, visit the `jx console` to check the status.
The first build will probably take a while, as the image for the gradle build pod
has to be downloaded etc.

To actually download and publish the zip, run the build with parameters.

1. Open the jenkins console (`jx console`)
1. Click on the `hybris-version-download` build job
1. Click on "Branches" on the top right
1. Click on the little Play-Icon (looks like `|>`) right next to the master branch
1. You will be prompted for the necessary parameters

If your build fails with `Required context class hudson.FilePath is missing`, you
probably forgot to configure the secrets `nexus` and `support-portal-user` in Jenkins!

**Important** You need to download and publish at least the latest hybris commerce 
6.7 patch version (6.7.0.2 as the time of writing) to build the Accelerator Demo (see below)
successfully

### Hybris Accelerator Demo Project

https://github.com/hybris-jenkins-x-demo/mycommerce

1. Fork the project into your own GitHub account or organization
1. Clone it to a local folder
1. `cd` into the clone
1. [optional] Change the maven nexus endpoint to your Jenkins-X installation in `build.gradle`, if you want to 
  build locally
1. Change the `ORG` name in `Jenkinsfile` and in `charts/mycommerce/Makefile` to the account / organisation you used in the first step
1. Change the `hybris.version` property in `build.gradle` to the latest patch release
1. commit and push any changes to branch `master`
1. run `jx import --no-draft --no-jenkinsfile` and follow the prompts to add to project to your Jenkins-X installation

## Setup Environments

After importing all the demo projects, we have a full CI pipeline for hybris commerce available inside our Kubernetes cluster.

Jenkins-X uses the concept of GitOps to install/deploy software to a so-called
environment.

Check the documentation of Jenkins-X about [environments](jx-env) and [promotion](jx-promote) for some background.

The steps below assume the env var `NEWENV` exists in the current shell session and
contains the ID of the new environment you want to create.
(e.g. `export NEWENV=develop`)

1. Create a new database for the environmet

        gcloud sql databases create $NEWENV --instance=hybris-database-test

1. Create a new MySQL user for the environment (make sure to use some better password)

        gcloud sql users create $NEWENV 'cloudsqlproxy~%' --instance=hybris-database-test --password=secret-password

1. Create new Kubernetes namespace for environment

        kubectl create namespace $NEWENV

1. Create secrets in the new namespace for the Cloud SQL connection

    Username and password from Step 2:

        kubectl create secret generic cloudsql-db-credentials \
        --from-literal=username=$NEWENV --from-literal=password=secret-password \
        --namespace=$NEWENV

    Service Account Secret (from Cloud SQL Setup)

        kubectl create secret generic cloudsql-instance-credentials \
        --from-file=credentials.json=$PROXY_KEY_FILE_PATH \
        --namespace=$NEWENV


1. Create a jx environment

        jx create env \
        --name $NEWENV \
        --label $NEWENV \
        --namespace $NEWENV \
        --fork-git-repo 'https://github.com/hybris-jenkins-x-demo/environment-template.git'

    For the `develop` environment, I would choose the `Auto` promotion strategy, 
    so every new release of the Demo Project is automatically deployed.

    Also, a new helm release named `develop` will be deployed after the pipeline
    finishes. (Verify by running `helm list` from the command line)

1. Clone the environment repository and adapt the template config

    - Search for the string `change-me` and replace it with the environment name
    - `env/values.yaml`: Adapt the endpoint URLs (`...XXX.nip.io`) with the correct one.

        The correct domain is available in the property `expose.config.domain` (auto-generated by Jenkins-X)

    - JDBC connection string (`y_db_url`) - 
      Change the instance configuration at the end to your Cloud SQL instance (`cloudSqlInstance=`).

      The Instance connection name can be found in the Cloud SQL console (Click on your instance, check the field "Instance connection name" under "Connect to this instance")

    After you are done, commit and push the changes.

1. Wait for the environment pipeline to finish (check the build logs with `jx get build log -f $NEWENV`)

1. Delete the helm release

        helm delete --purge $NEWENV

    The helm lifecycle plugins of the Accelerator Demo helm charts work only for
    a fresh release (especially the initialization of a new hybris environment)

1. Promote the Accelerator Demo Project to the new environment

        cd <demo-project-repository>
        jx promote --env $NEWENV --version 0.0.1

    **Make sure to delete the auto-deployed helm release before (see previous step)**
    
    Version 0.0.1 should be the first version of your fork. Check the git tags of
    your fork of the Accelerator Demo for all available versions

    The command above creates a pull request with the changes to the environment.
    Depending on the promotion strategy, you may have to manually approve the pull
    request in GitHub.

    If you promote the Accelerator Demo to a brand new environment, the initialization
    takes about 20min (Create new Kubernetes resources + `ant initialize`)

## Troubleshooting

### Git related errors (merge conflicts etc.) when promoting the Accelerator Demo to an environment using the command line

`jx` keeps its own local clone of the environment git repository in `~/.jx/environments/...`.

Deleting those folders and trying again should resolve any issues.

### Monitor helm lifecycle hooks

    # switch kubernetes namespace to environment you want to monitor
    jx namespace $ENVIRONMENT

    # Monitor system initialization (ant initialize)
    kubectl logs -f $(kubectl get pods -l "job-name=$ENVIRONMENT-mycommerce-init" -oname)

    # Monitor rolling update (ant updatesystem)
    kubectl logs -f $(kubectl get pods -l "job-name=$ENVIRONMENT-mycommerce-update" -oname)

### Unschedulable $ENVIRONMENT-mycommerce-update job

You probably promoted a new environment without deleting the initial helm release first.

1. Kill the faulty helm release (ignore any errors regarding missing resources)

        helm delete --purge $ENVIRONMENT
        kubectl delete job $ENVIRONMENT-mycommerce-update --namespace $ENVIRONMENT
        kubectl delete configmap $ENVIRONMENT-mycommerce --namespace $ENVIRONMENT

    Also, check the GKE Console for any release-related resources still remaining.

1. Re-run the Jenkins job for the environment repository (usually called `environment-jx-$ENVIRONMENT`)

[gcloud-install]: https://cloud.google.com/sdk/docs/downloads-interactive
[gcloud-init]: https://cloud.google.com/sdk/docs/initializing
[gsql-quickstart]: https://cloud.google.com/sql/docs/mysql/quickstart
[jx-install]: https://jenkins-x.io/getting-started/install/
[gke-multi]: https://cloud.google.com/kubernetes-engine/docs/concepts/multi-zone-and-regional-clusters
[helm]: https://helm.sh/
[helm-rbac]: https://docs.helm.sh/using_helm/#role-based-access-control
[nfs-provisioner]: https://github.com/kubernetes/charts/tree/master/stable/nfs-server-provisioner
[nginx]: https://kubernetes.github.io/ingress-nginx/
[cert-manager]: https://cert-manager.readthedocs.io/en/latest/
[jx-env]: https://jenkins-x.io/about/features/#environments
[jx-promote]: https://jenkins-x.io/developing/promote/