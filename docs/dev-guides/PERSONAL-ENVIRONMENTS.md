# Personal Test Environments
Personallly provisioned test environments, and the steps to create and manage them, are described here.
## Background
### Use Case
Personal Kubernetes environments allow realistic testing of deployed services without interfering
with shared environments. This enables faster development cycles, as code can be tested without
waiting for a release, and without risk of breaking colleagues' environments.

### Where they Live
A personal environment is contained by a K8s [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
named for a developer, sharing a larger cluster in the GCP
project [terra-kernel-k8s](https://console.cloud.google.com/home/dashboard?project=terra-kernel-k8s)
called 
[terra-integration](https://console.cloud.google.com/kubernetes/clusters/details/us-central1-a/terra-integration?project=terra-kernel-k8s&tab=details&persistent_volumes_tablesize=50&storage_class_tablesize=50&nodes_tablesize=50&node_pool_tablesize=10).
Deployments target this namespace and allow `kubectl` commands to be narrowly focused. The deployment
consists of a single
[pod](https://kubernetes.io/docs/concepts/workloads/pods/) and [service](https://kubernetes.io/docs/concepts/services-networking/service/)
in the case of Workspace Manager, though the system is more flexible than that.

### Helpful Background Reading
#### Kubernetes
[The Kubernetes Book]() is very accessible for beginners (like myself), and uses lots of images and
repetition to bring the main points home. Additionally, the exercises on the main Kubernetes site are
well-paced, insightful, and a realistic facsimile of a k8s cluster and the process of interacting
with it. The [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) has lots
of useful goodies.

#### Terraform
[Terraform Up & Running](https://www.amazon.com/Terraform-Running-Writing-Infrastructure-Code/dp/1491977086)
is an excellent resource for getting your head around Terraform and some of the
design decisions that went into it.

## Prerequisites
### Accounts
You'll need
1. An `@broadinstitute.org` account
2. Membership in the Google group [dsde-engineering](https://groups.google.com/a/broadinstitute.org/g/dsde-engineering)
for access to private Github repos and [ArgoCD](https://argoproj.github.io/argo-cd/)
3. A public GitHub account
4. A linkage between this GitHub account and your Broad account, using the [Broad Institute GitHub App](https://broad-github.appspot.com/). 
5. (helpful) SSH credentials for using GitHub
6. Membership in the private Git team for Platform Foundations.
7. A first initial and last name. I'm using `jcarlton` throughout so I don't
have to edit commands and outputs from my sessions.

For most permissions issues, contact DevOps.

## Step 1: Terraform Orchestration for Terra Environment
At this point, the [existing documentation](https://docs.dsp-devops.broadinstitute.org/mc-terra/mcterra-quickstarts#Create-a-new-namespaced-environment-in-an-existing-cluster)
in the DevOps Confluence is applicable, and some of the following borrows from it. The process involves
two different pull requests in different repositories. The first is a change to the 
[terraform-ap-depoyments](https://github.com/broadinstitute/terraform-ap-deployments)
repo, and the subject is any cloud dependencies that must be instantiated for the new environment. Different
use cases will have different needs here, but for Workspace Manager we're interested in the following.

The Broad's Terraform modules live in a public repository called [terraform-ap-modules](https://github.com/broadinstitute/terraform-ap-modules).
These are combined with generated private client modules, variables, and secrets to instantiate the
actual environments and artifacts. The latter are contained in the private repo [terraform-ap-deployments](https://github.com/broadinstitute/terraform-ap-deployments), 
and this is the one we need to change. See this [DevOps Portal entry](https://docs.dsp-devops.broadinstitute.org/mc-terra/mcterra-deployment#Infrastructure---Terraform)
for more background.

First, we declare a new Terra Environment in [atlantis.yaml](https://github.com/broadinstitute/terraform-ap-deployments/blob/master/atlantis.yaml#L70). A new object
in a yaml array describes the top-level properties of the environment. Note that "environment" here
is a set of conventions for creating a stand-alone instance of one or more services, and not anything
specific in k8s or Terraform. Mine looks like this:
```yaml
- name: terra-env-jcarlton
  dir: terra-env
  workflow: terra-env-jcarlton
  workspace: jcarlton
  autoplan:
    enabled: false
```
It's easy to get confused between the env name (`terra-env-jcarlton`), the workspace (`jcarlton`), and the k8s namespace eventually
created (`terra-jcarlton`). There are numerous examples of the conventions in this file, and I admit
I cheated a little off the surrounding entries. Note that Atlantis is a tool for managing Terraform updates,
but atlantis.yaml is not itself a Terraform file.

Second, we specify some workflow settings for deploying the environment. Among other things, this tells
Atlantis to pass the variables file `jcarlton.tfvars`
```yaml
  terra-env-jcarlton:
    plan:
      steps:
        - init:
            extra_args: ["-upgrade"]
        - plan:
            extra_args: ["-var-file", "tfvars/jcarlton.tfvars"]
```

Third, we check in a tfvars file for Terraform to use as its input variables when building this [workspace](https://www.terraform.io/docs/state/workspaces.html).
```hcl-terraform
# General
google_project = "terra-kernel-k8s"
cluster        = "integration"
cluster_short  = "integ"

terra_apps = {
  workspace_manager = true,
  crl_janitor       = true,
  buffer            = true,
}

# Resource Buffer Service Testing
buffer_google_folder_ids   = ["637867149294"]
buffer_billing_account_ids = ["01A82E-CA8A14-367457"]

# Workspace Manager
wsm_workspace_project_folder_id = "1094831421623" # DSP Integration
wsm_billing_account_ids         = ["01A82E-CA8A14-367457"]
```
This file should be in a path like `terraform-ap-deployments/terra-env/tfvars/jcarlton.tfvars`. IDs will vary, thougth I borrowed
mine from a teammate. Folder IDs refer to [GCP Project Folders](https://cloud.google.com/resource-manager/docs/creating-managing-folders).
The entries under `terra_apps` describe services your deployment will depend on (but not deploy). Google project and cluster
information should be the same for everyone.

The repository is integrated with [Atlantis](https://www.runatlantis.io/), which checks terraform artifacts
before merge to make sure they can `plan` and `apply` successfully. When pushing a PR to this repo, 
it's expected you'll run `atlantis` commands as part of the review process. First, a github comment of
 ```shell script
 atlantis plan -p terra-env-jcarlton
```
to which `broadbot` will reply:
```text
Ran Plan for project: terra-env-jcarlton dir: terra-env workspace: jcarlton

Show Output
```

Following a successful plan, run `atlantis apply`, and if that's successful (and you get an approval), 
you can merge the PR.
## Step 2: Edit the Helm File and Auxiliary Files
[Helm](https://helm.sh/) manages k8s packages called [Helm Charts](https://helm.sh/docs/topics/charts/),
allowing large-scale Kubernetes deployments. These may be published in public or private repositories,
similarly to Docker container applications. In turn, a [helmfile](https://github.com/roboll/helmfile)
may be used to work with many charts in a large, heterogeneous application (like Terra).

### Aside: look through the Charts
The main Terra Helm charts don't need changing for our deployment, but we set up instructions to create
an instance with it. Charts for Terra are located in the public [terra-helm repo](https://github.com/broadinstitute/terra-helm).
The chart directory for workspace manager is [here](https://github.com/broadinstitute/terra-helm/tree/master/charts/workspacemanager).
THe chart is actually surprisingly brief, showing only one dependency.
```yaml
apiVersion: v2
name: workspacemanager
version: 0.19.0
description: Chart for Terra Workspace Manager
type: application
keywords:
  - dsp
  - broadinstitute
  - terra
  - workspacemanager
sources:
  - https://github.com/broadinstitute/terra-helm/tree/master/charts
  - https://github.com/DataBiosphere/terra-workspace-manager
dependencies:
  - name: postgres
    condition: postgres.enabled
    version: 0.1.0
    repository: https://broadinstitute.jfrog.io/broadinstitute/terra-helm-proxy
```

It's also possible to navigate the helm repo itself using the `helm` command:
```shell script
$ helm repo add terra-helm https://broadinstitute.github.io/terra-helm
```
```shell script
$ helm repo update
```
```text
...Successfully got an update from the "terra-helm" chart repository
```

Then just do 
```shell script
$ helm pull terra-helm/workspacemanager
```
and a .tgz file such as `workspacemanager-0.19.0.tgz` should be downloaded silently to the current directory.

We don't need to make any changes to the `terra-helm` repo in order to set up a personal environment.

### The Helmfile
The helmfile allows specifying override values for a chart for a particular
deployment, and the [terra-helmfile repo](https://github.com/broadinstitute/terra-helmfile)
is the place providing and for editing those. This "gitOps" model
is similar to the relationship between the private, Atlantis-enabled `terraform-ap-deployments` repo
and the public `terraform-ap-modules` that we saw above.

We don't actually need `helm` installed locally for this step, though it will be required later.


Clone the [terra-helmfile](https://github.com/broadinstitute/terra-helmfile) and look at the file structure.
```shell script
$ tree -L 2
```

```
.
├── README.md
├── argocd
│   └── values
├── bin
│   ├── logging.sh
│   └── render
├── clusters.yaml
├── docs
│   └── NAMING.md
├── envDefaults.yaml
├── environments
│   ├── live
│   ├── live.yaml
│   ├── personal
│   ├── personal.yaml
│   └── preview.yaml
├── helmfile.yaml
├── terra
│   └── values
├── versions
│   ├── README.md
│   ├── alpha.yaml
│   ├── prod.yaml
│   └── staging.yaml
└── versions.yaml

```
You can see the main `helmfile.yaml`, an `environments` directory with subdirectories for `live` and `personal`, and a `terra` directory
with subdirectories for each service:
```shell script
jaycarlton~/repos/terra-helmfile/terra (master) $ tree -L 2
```

```
.
└── values
    ├── agora
    ├── buffer
    ├── buffer.yaml
    ├── consent
    ├── crljanitor
    ├── crljanitor.yaml
    ├── cromiam
    ├── cromwell
    ├── datarepo
    ├── datarepomonitoring
    ├── duos
    ├── firecloudorch
    ├── global
    ├── global.yaml
    ├── hydra
    ├── icdemo
    ├── identityconcentrator
    ├── istioconfig
    ├── istioconfig.yaml
    ├── leonardo
    ├── ontology
    ├── poc
    ├── poc.yaml
    ├── rawls
    ├── sam
    ├── workspacemanager
    └── workspacemanager.yaml
```
For this PR we'll edit all of
* the helmfile itself, to make it aware of our personal enviornment
* a personal.yaml file defining our environment's enabled services at a  high level.
* several values files defining parameters for the individual services in the environment

### Helmfile
The main helmfile itself gets a simple two-line entry for this project pointing to other files with
properties to read (including the one above):
```yaml
  jcarlton:
    values:
      - environments/personal.yaml
      - environments/personal/jcarlton.yaml
```

### Personal yaml file
This file describes the Terra apps that are required for the personal environment. In my case those were
resource buffer, janitor, and workspace manager. For the file `terra-helmfile/environments/personal/jcarlton.yaml`:
```yaml
# Environment overrides for jcarlton environment
releases:
  crljanitor:
    enabled: true
  workspacemanager:
    enabled: true
  buffer:
    enabled: true
```
If any environments are disabled, their resources are not generated by Terraform.

The main personal.yaml is nearly empty (and should not be edited). It appears just to be a lightweight
bit of plumbing so the whole repo isn't just one massive file.
<details>

<summary>personal.yaml</summary>

```yaml
# Common settings for private developer environments
base: 'personal'

# All private dev environments go to same cluster
defaultCluster: terra-integration

# Enable services below, eg:
# releases:
#   cromwell:
#     enabled: true
```

</details>

Finally, each service to be included has its own `values` override file for our environment:
For `terra/values/workspacemanager/personal/jcarlton.yaml`, you should verify that [vault](https://www.vaultproject.io/) has secret
stored at the path described. In my case this had already been taken care of.

```yaml
vault:
  pathPrefix: secret/dsde/terra/kernel/integration/jcarlton

ingress:
  staticIpName: terra-integration-jcarlton-workspace-ip
```

For `terra/values/crljanitor/personal/jcarlton.yaml`, you need the IP address from the `atlantis apply` call in the previous PR.
```yaml
vault:
  pathPrefix: secret/dsde/terra/kernel/integration/jcarlton
  configPathPrefix: config/terraform/terra/integration/jcarlton
serviceIP: 35.223.31.120
```

For `buffer`, there are additional properties:
```yaml
vault:
  pathPrefix: secret/dsde/terra/kernel/integration/jcarlton

ingress:
  staticIpName: terra-integration-jcarlton-buffer-ip

buffer:
  pool:
    configPath: config/toolsalpha

proxy:
  whitelist:
    email: "buffer-client-jcarlton@terra-kernel-k8s.iam.gserviceaccount.com"
```
## Creating the namespace
### `kubectl` Setup
Interaction with Kubernetes is most often accomplished with `kubectl`. This is a local utility that is packaged
with Kubernetes. Type `kubectl version` to make sure it's installed.
```shell script
$ kubectl version
```
<details>

```text
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"17+", GitVersion:"v1.17.14-gke.400", GitCommit:"b00b302d1576bfe28ae50661585cc516fda2227e", GitTreeState:"clean", BuildDate:"2020-11-19T09:22:49Z", GoVersion:"go1.13.15b4", Compiler:"gc", Platform:"linux/amd64"}
```

</details>

 To connect `kubectl` to the cluster, you must first update `kubeconfig` with the proper credentials. 
`gcloud container` has a [clusters get-credentials](https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials) option to support exactly this use case. In our case
we need to specify both the project and zone as well as the cluster name:  
```shell script
$ gcloud container clusters get-credentials terra-integration --zone us-central1-a --project terra-kernel-k8s
```
```text
Fetching cluster endpoint and auth data.
kubeconfig entry generated for terra-integration.
```

### Making the namespace
```shell script
$ kubectl create namespace terra-jcarlton
```
```text
namespace/terra-jcarlton created
```

##  Deploying (WorkspaceManager-specific)
The Workspace Manager repository has a [dev-local directory](https://github.com/DataBiosphere/terra-workspace-manager/tree/dev/local-dev)
with instructions in its README for loading the helm chart and terraform state onto a pod in the new namespace. 

## Syncing in ArgoCD
The [terra-app-generator](https://ap-argocd.dsp-devops.broadinstitute.org/applications/terra-app-generator)
application in ArgoCD needs to sync the new environment and its dependent sources. As of this writing,
only a member of DevOps engineers and admins have access to this app. So contact someone in #dsp-devops-champions and ask to have your new app synced.
## Using `kubectl` 
To view the current `kubectl config` do
```shell script
$ kubectl config view
```

<details>

<summary>Config view output</summary>

```text
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://34.70.205.77
  name: gke_terra-kernel-k8s_us-central1-a_terra-integration
contexts:
- context:
    cluster: gke_terra-kernel-k8s_us-central1-a_terra-integration
    namespace: terra-jcarlton
    user: gke_terra-kernel-k8s_us-central1-a_terra-integration
  name: gke_terra-kernel-k8s_us-central1-a_terra-integration
current-context: gke_terra-kernel-k8s_us-central1-a_terra-integration
kind: Config
preferences: {}
users:
- name: gke_broad-dsde-dev_us-central1-a_terra-dev
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /Users/jaycarlton/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
- name: gke_terra-kernel-k8s_us-central1-a_terra-integration
  user:
    auth-provider:
      config:
        access-token: <<REDACTED>>
        cmd-args: config config-helper --format=json
        cmd-path: /Users/jaycarlton/google-cloud-sdk/bin/gcloud
        expiry: "2021-01-21T19:54:04Z"
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```
</details>

The credentials and other info are contained in a `kubectl context` and can be inspected:
```shell script
kubectl config get-contexts
```
```text
CURRENT   NAME                                                   CLUSTER                                                AUTHINFO                                               NAMESPACE
*         gke_terra-kernel-k8s_us-central1-a_terra-integration   gke_terra-kernel-k8s_us-central1-a_terra-integration   gke_terra-kernel-k8s_us-central1-a_terra-integration   terra-jcarlton
```
### Getting Descriptions
To get the description of the namespace, use `describe namespace`
```shell script
$ kubectl describe namespace terra-jcarlton
```
```text
Name:         terra-jcarlton
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:                       gke-resource-quotas
 Resource                    Used  Hard
 --------                    ---   ---
 count/ingresses.extensions  1     5k
 count/jobs.batch            0     10k
 pods                        1     5k
 services                    1     1500

No LimitRange resource.
```

The output shows that I have an active namespace with no labels and I'm well under quota. To view details
about the pods, use `get pods` and isolate the namespace to use:
```shell script
$ kubectl get pods --namespace=terra-jcarlton
```
```text
NAME                                           READY   STATUS     RESTARTS   AGE
workspacemanager-deployment-68dfc86fbb-zhpwc   0/3     Init:0/1   0          4h35m
```
 For the service, use `get services`
 ```shell script
$ kubectl get services --namespace=terra-jcarlton
```
```text
NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
workspacemanager-service   ClusterIP   10.56.8.103   <none>        443/TCP   4h39m
```

It's also possible to set this namespace as a default, at which point you can leave off the `--namespace`
qualifier:
```shell script
$ kubectl config set-context --current --namespace=terra-jcarlton
```
```text
Context "gke_terra-kernel-k8s_us-central1-a_terra-integration" modified.
```

To list the nodes in the cluster do
```shell script
kubectl get nodes
```

<details>

<summary>Node list</summary>

```text
NAME                                             STATUS   ROLES    AGE   VERSION
gke-terra-integration-cronjob-v1-9d4ae6cb-dvz6   Ready    <none>   31h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-hv62      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-jrmn      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-m2ui      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-pa8n      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-qstb      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-c1ae3b79-rert      Ready    <none>   30h   v1.17.14-gke.400
gke-terra-integration-default-v2-95d71f95-gdfy   Ready    <none>   30h   v1.17.14-gke.400
```

</details>