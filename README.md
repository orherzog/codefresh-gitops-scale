# codefresh-gitops-scale

Notes that I summerized from the Codefresh GitOps Fundamentals Course

## Table of Contents

- [The App of Apps pattern](#the-app-of-apps-pattern)
- [Multi-cluster management](#multi-cluster-management)
- [Generating applications with ApplicationSets](#generating-applications-with-applicationsets)
- [Git Repository Strategies](#git-repository-strategies)
- [Promoting the application using CI](#promoting-the-application-using-ci)
- [Argo CD Image Updater](#argo-cd-image-updater)
- [Sync Phases/Hooks](#sync-phaseshooks)
- [Sync Waves](#sync-waves)
- [Sync and Diff Strategies](#sync-and-diff-strategies)

## The App of Apps pattern

There are two main ways to handle multiple applications, App of Apps, and Application Sets.

Argo CD allows us to define a root Argo CD application that will itself define and sync multiple child applications. The Root App points to a folder in Git, where we store the application manifests that define each application we want to create and deploy.

The main advantage of using App of Apps is that you can now treat several applications as a single unit while still keeping them isolated during deployment.

The directory structure will look like the following: 
```bash
My Apps

├── my-projects

│   ├── my-project.yml

├── my-apps

│   ├── child-app-1.yml # The source path for the child apps point to a path where we defince the manifests of our applications using any of the supported ways (Helm, Kustomize, plain YAMLs etc).

│   ├── child-app-2.yml

│   ├── child-app-3.yml

│   ├── child-app-4.yml

├── root-app # The root app folder where we define the root app

└───├── parent-app.yml # The source path is points to the folder of the other child applications
```

Each Argo CD application monitors all Kubernetes components it manages. In a similar way, each root application monitors all its child application definitions. The root application can monitors the health and will detect any discrepancy on the child applications. It will optionally resync them if you choose to do so (automatic sync, self-heal enabled).

Apart from deploying all applications at once you can also remove them by removing the parent application, after that all the child applications will be removed and the cluster.

* Note - The Argo CD application need to be installed on the argocd namespace so that Argo CD can detect it. 

* Tip: 
    In order to install your root app from the public repository you can just using kubectl apply command using the following command:
    
```bash
    kubectl apply -f https://raw.githubusercontent.com/<your user>/gitops-cert-level-2-examples/main/app-of-apps/root-app/my-application.yml -n argocd
```

## Multi-cluster management

Argo CD has the capability to connect and deploy to external clusters.

To add one of your public/local clusters with the argocd CLI, you first need to ensure you have a valid context in your kubeconfig for the cluster.

### Add new cluster to Argo
```bash
 argocd cluster add <context name>
```
This command will create a new ServiceAccount (called argocd-manager) into the kube-system namespace of your kubectl context and binds the service account to an admin-level ClusterRole on the target cluster.

The configuration and connection information is added to the management cluster in the form of a secret.

### Remove cluster from Argo
```bash
argocd cluster rm <server> # The server will be the cluster server URL, you can get it using argocd cluster list
```
Notice that when removing a cluster it does not remove the Argo CD Applications associated with it.


## Generating applications with ApplicationSets

An ApplicationSet uses templated automation to create, modify, and manage multiple Argo CD applications at once, while also targeting multiple clusters and namespaces.

The ApplicationSet controller is installed alongside Argo CD (within the same namespace) and creates multiple Argo CD Applications based on the ApplicationSet Custom Resource (CR).

Application sets can be used in several scenarios such

* Deploying a single application to many namespaces
* Deploying a single application to many clusters
* Deploying many applications in a single cluster
* Deploying many applications in many clusters

The ApplicationSet is made up of “generators” that generate Applications, they are responsible for providing a set of key-value pairs, that are then passed into a template with {{param}} styled parameters.

The template fields within an ApplicationSet spec are used to generate an Argo CD Application resource. The Argo CD Application is created by combining the params from the generator with the fields from the template.

```bash
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://kubernetes.default.svc
      - cluster: engineering-prod
        url: https://kubernetes.default.svc
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/applicationset.git
        targetRevision: HEAD
        path: examples/list-generator/guestbook/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
```

Within the example above the List generator is passing the {{cluster}} and {{url}} fields into the Application template as the parameters.

These fields will be rendered as two corresponding Argo CD Applications - one for each defined cluster.

```bash
spec:
  generators:
  - list:
      elements:
      - cluster: engineering-dev
        url: https://kubernetes.default.svc
      - cluster: engineering-prod
        url: https://kubernetes.default.svc
```

### Generators

A Generator informs your ApplicationSet on how to generate multiple Applications and how to deploy them.

* List Generator - This is the Generator used to generates parameters based on a fixed list.
* Cluster Generator - For each of your clusters you’ve registered with Argo CD, the Cluster Generator will produce your params based on the list of values within the cluster secret.
* Git Generator -  Includes two sub-types:
    * Directory Generator - Generates params using the directory structure of your specific Git repository.
    * File Generator - generates params using the content within a JSON/YAML file in your Git repository and reads a configuration file.
* SCM provider generator - Discover and iterate over whole Git repositories that are found in your provider (e.g. GitHub). This can be used for completely automating the creation of an environment and/or application whenever somebody creates a new repository in the specified GitHub organization.
* Pull Request Generator - Iterate on Pull Requests from your Git provider (e.g. GitHub) and then create environments for each pull request found.
* Cluster Decision Resource Generator - Discover and list clusters of ArgoCD based on specific Kubernetes resources of any type that satisfy a set of criteria.
* Matrix generator - Used for combining the primary generators. For example if you already have a Cluster generator for all your clusters and Git generator for all your apps, you can combine them with a matrix generator to deploy all your apps to all your clusters.

Examples - 

Directory Generator - The name of the application is generated based on the name of the directory and denoted as the param: {{path.basename}} within the config in the YAML file below.

```bash
  generators:
  - git:
      repoURL: https://github.com/hseligson1/appset-demo.git
      revision: HEAD
      directories:
      - path: examples/git-dir-generator/apps
  template:
    metadata:
      name: '{{path[0]}}'
```

File Generator -

A config.json is a file that contains the information on how to deploy the applicationm for environment-specific clusters. Each folder contains the configuration file that describes the cluster.

```bash
├── apps
│   └── app-1
│       ├── templates
│       ├── Chart.yaml
│       └── values.yaml
├── cluster-config
│   └── environments
│       ├── staging
│       │   └── config.json
│       └── prod
│           └── config.json
└── git-generator-files.yaml
```
Included within the directory structure are:

* app-1: an application
* cluster-config: this contains JSON/YAML files that describe environment-specific clusters: one for staging and prod. Each folder contains the configuration file that describes the cluster.
* git-generator-files.yaml: this is the application resource used to deploy our app-1 application to the clusters.

Whenever a change is made to the config.json file within the cluster-config folder above, it will automatically be discovered by the git-generator.

## Git Repository Strategies

There are four distinct categories of suggested solutions:

* Environment-per-branch - In this case there is a Git branch for each environment (Not recommended).
* Environment-per-folder - In this case all environments are in a single Git repository and all are in the same branch. The filesystem has different folders that hold configuration files for each environment (Most recommended).
* Environment-per-repository - In this case each environment is on its own git repository.
* Combination of the approaches - Combination of any of the previous approaches.

By using folders/overlays for different environments you can promote any release to any other environment by simple file copying. Also, We can automate promotions by using a CI system (such as Github actions).

## Promoting the application using CI

Automating promotions is a done by copying files can be automated by your CI system or any other high level system.

## Argo CD Image Updater

Argo CD Image Updater is a tool that continuously monitors new container images in Kubernetes workloads managed by Argo CD. Image Updater enables a way for you to override your image tag for your Argo CD application allowing you to dynamically determine values for your parameters for your image version without needing to manually change your deployment manifest.

Argo CD Image Updater does this by watching your container registries, so whenever a new version is available, the Image Updater will commit an override file in your repository. 

Image Updater can only update container images for applications with manifests that are rendered using templating tools such as Helm or Kustomize.

There are 2 methods to install the Argo CD Image Updater - 

* Method 1: Installing the Image Updater as a Kubernetes workload in the Argo CD namespace
* Method 2: Connect using the Argo CD API Server

### Installing Image Updater as Kubernetes Workload in the argocd Namespace

This method allows you to install the image updater as a Kubernetes workload within the namespace.

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

## Sync Phases/Hooks

When an application has multiple Kubernetes resources Argo CD will by default handle their deployment in the order that makes the most sense.

Sometimes however, you need to override the default order of components and define a specific sequence of resources that should be applied.

To assign a resource to a specific phase you need to use the argocd.argoproj.io/hook annotation. 

Hooks can be any type of Kubernetes resource kind, but tend to be Pod, Job or Argo Workflows. Multiple hooks can be specified as a comma separated list.

## Sync Waves

Argo CD also offers an alternative method of changing the sync order of resources. These are sync waves. 

They are defined by the argocd.argoproj.io/sync-wave annotation. The value is an integer that defines the ordering (ArgoCD will start deploying from the lowest number and finish with the highest number).

Sync waves are a number you assign in each resource. By default is each resource is assigned to wave 0 but you can add both negative and positive values to different resources.

During a deployment ArgoCD will order all resources from the lowest number to the highest and then deploy them according to that order.


```bash
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: '5'
```

When a sync operation takes place, Argo CD will:

* Order all resources according to their wave (lowest to highest)
* Apply the resources according to the resulting sequence

There is currently a delay between each sync wave in order to give other controllers a chance to react to the spec change that was just applied. This also prevents Argo CD from assessing resource health too quickly (against the stale object), causing hooks to fire prematurely. The current delay between each sync wave is 2 seconds and can be configured via the environment variable ARGOCD_SYNC_WAVE_DELAY.

When Argo CD starts a sync, it orders the resources in the following precedence:

* The phase
* The wave they are in (lower values first)
* By kind (e.g. namespaces first and then other Kubernetes resources, followed by custom resources)
* By name

### Using the PreSync phase

One of the most common scenarios for application deployment is a task that needs to run before the main sync phase. An example would be a database upgrade with a new schema.

Use the PreSync phase when you want a resource to be deployed BEFORE the main sync process

```bash
 argocd.argoproj.io/hook: PreSync
```

### Deploying resources after a sync

Similar to the PreSync phase, you can also deploy resources in the PostSync phase which ArgoCD will run after the main sync phase is complete.

```bash
 argocd.argoproj.io/hook: PostSync
```

### Running a job on sync failure

Apart from the PreSync, Sync and PostSync phases, ArgoCD also has an optional SyncFail phase that is only activated if the main Sync phase has any errors.

In the case Argo CD will apply/deploy all resources that are marked with the SyncFail annotation. Notice also that a failure in the Sync process will never execute any PostSync resources.

## Sync and Diff

Argo CD introduced a new sync option to verify diffing customizations while preparing a JSON patch to be applied in the cluster and JQ path expression, as well as leverage the metadata.managedFields within the live resources, to ignore differences found in the spec fields. Both of these approaches leverage managedFields metadata to instruct Argo CD about the trusted managers and fields to ignore that are controlled by those managers.

```bash
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

## Notifications

Argo CD comes with a notification engine that allows you to intercept various ArgoCD events and send them to external systems. This notification engine is also used by Argo Rollouts.

It is also worth mentioning that the notification engine is to be used only in simple scenarios. If you have a complex use case it is better to use a dedicated solution such as Argo Events and optionally Argo Workflows.

## Sync windows

There are cases where you want to guarantee that no sync operation will take place either automatically or by human intervention at a specific time period.

Some common examples are:

* No deployments should ever happen on Sunday
* Deployments should only happen during working hours/days
* Deployments should only happen every Monday and Thursday
* Deployments should only happen at night at 4.00 am

Sync windows are time periods that you define where you can explicitly disable/enable application syncing. Sync operations will only be possible if the active syncing windows allow it.


```bash
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
spec:
  syncWindows:
    - kind: allow
      schedule: 10 1 * * *
      duration: 1h
      applications:
        - '*-prod'
      manualSync: true
    - kind: deny
      schedule: 0 22 * * *
      duration: 1h
      namespaces:
        - default
        - kind: allow
          schedule: 0 23 * * *
          duration: 1h
          clusters:
            - in-cluster
            - cluster1
```