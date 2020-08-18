# GitOps with Flux

This is an extension to the the previous [post](ci-containers.md) on CI with Containers. In this post we will extend the last step of the deployment to save the manifest files in a Git repository.

## GitOps

[GitOps](https://www.gitops.tech/) is a way of implementing Continuous Deployment for cloud native applications

- Extension to the CICD process
- Cloud Native Continuous Delivery
- Resource management and provisioning is declarative
- Orchestrated and repeatable pattern
- Git (version control) as the source of truth
- Deployment can be version controlled

### Benefits

- Single tool and interface for controlling your infrastructure
- Version control for all the changes
- Easy rollbacks and audits
- Perform diff on deployments
- Git push is a close to developers
- Inherent security benefits

### Installation

There are few different ways of getting flux installed and we will use Helm for this article. Below outlines the high level steps:

1. *FluxCtl* [installation](https://docs.fluxcd.io/en/1.17.0/references/fluxctl.html#linux): the cli to manage flux
2. *Flux Daemon* installation: Installation within the kubernetes cluster
3. *Git* repo: the repository to house the manifest
4. *Identity*: Grant the Daemon access to the Git repo

**Flux Daemon Installation**
```bash
    helm repo add fluxcd https://charts.fluxcd.io
    kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml

    kubectl create namespace flux

    helm upgrade -i flux fluxcd/flux --set git.url=git@ssh.dev.azure.com:v3/orgname/projectname/static-web-app-gitops --namespace flux --set manifestGeneration=true

    helm upgrade -i helm-operator fluxcd/helm-operator    --set git.ssh.secretName=flux-git-deploy    --namespace flux --set helm.versions=v3
```

**Git repo**

Once the deamon is installed, we need a repo to be the source of truth for the manifest files. Flux is capable of manifest generation through [Kustomize](https://kustomize.io/) which is a template-free way to define manifest across environments.

Fork the sample [repo](https://github.com/seenu433/mvc-static-web-gitops)

**Identity**

To grant Flux Daemon access to the git repo, generate an SSH key using the cli and add it to the SSH keys in User Settings -> SSH Public Keys.

```bash
    fluxctl identity --k8s-fwd-ns flux
```
## Pipeline Integration

Once we have the repo setup, we will extend the pipeline in the previous [post](ci-containers.md) by modifying the **Deployment** section of the script with the below:

```bash
    
    # Deployment
    #GitOps

    # clone the gitops repo
    git clone https://username:PAT@dev.azure.com/orgname/projectname/_git/mvc-static-web-gitops
    
    # update the yamls
    kubectl create configmap css-file --from-file=site.css -n devops-demo --dry-run=client -o yaml > static-web-app-gitops/workloads/backend/configmap.yaml
    
    cat deploy.yaml > static-web-app-gitops/workloads/backend/deployment.yaml
    
    cd static-web-app-gitops/workloads/backend
    
    # push the changes
    git config  user.email "email@company.com"
    git config  user.name "user"
    
    git add *
    
    git commit -m  "comments"
    
    git push https://username:PAT@dev.azure.com/orgname/projectname/_git/mvc-static-web-gitops

```

The pipeline will not publish the changes to the repo instead of a direct deployment to kubernetes.