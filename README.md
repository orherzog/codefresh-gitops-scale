# codefresh-gitops-scale

Notes that I summerized from the Codefresh GitOps Fundamentals Course

## Table of Contents


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

└───├── parent-app.yml # The source path of the app definition is my-apps directory
```