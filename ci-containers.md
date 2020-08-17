# Continuos Integration for Container Deployment

When interacting with customers on continuos integration for container deployment, I recommend discussing the below concepts:

1. Branching Strategy: [Gitflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) is a widely used strategy which i prefer to derive the image tags from the branch names.
2. Include the release number in the branch name.
3. Use the [BuildId](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml) or [ReleaseId](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/variables?view=azure-devops&tabs=batch) to create a unique tag for every build/release.
4. If deploying to Kubernetes, have the yaml templates in the repository.

## Scenario

Customer receives an image from a vendor into a repository and has to update the configuration specific to an environment.

Customer is building an image wrapping the vendor provided image and replacing the configuration file.

The recommendation is not to rebuild the image but override the configuration file in the container through mounts

- Mount Path configuration for [App Service](https://docs.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage?pivots=container-linux)
- Volume Mounts for [Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

To demonstrate this scenario, a simple [static web app](https://github.com/seenu433/mvc-static-web) is used to perform CI to update the background color in the css file without rebuilding the image.

```dotnetcli
docker build -t myregistry.azurecr.io/staicweb:latest .
docker push myregistry.azurecr.io/staicweb:latest
```

Now that we have an image ready, we will attempt to enable continuos deployment on change of configuration or on change of the vendor provided image

1. Create a new git repository with the deployment manifest and the configuration files (site.css in this case). [sample] (https://github.com/seenu433/mvc-static-web-ci)
2. Follow the Gitflow branching strategy by naming the release branch as release-version. For ex. release-4.1
3. Create a release pipeline with the artifacts as the ci git repo
4. Enable *Continuos Deployment Trigger* with Branch Filter to include all release branches **release/***
5. Add a *Bash Script* task to the deployment stage. Use the sample script explained below

### Sample Script

```bash

    # Build Image Tag
    echo $BUILD_SOURCEBRANCH
    echo $RELEASE_RELEASEID

    # Extract the release number from the branch name
    IFS='-'
    read -ra ADDR <<< "$BUILD_SOURCEBRANCH"
    echo ${ADDR[1]}

    # Append the releaseid to make a unique tag
    imageTag="${ADDR[1]}.$RELEASE_RELEASEID"

    echo $imageTag

    #Get password from key vault
    docker login srpadala.azurecr.io -u username -p password

    # Retag the vendor image with the release tag
    docker pull myregistry.azurecr.io/staicweb:latest
    tagcmd="docker tag myregistry.azurecr.io/staicweb:latest myregistry.azurecr.io/staicweb:$imageTag"
    echo $tagcmd
    eval "$tagcmd"
    pushcmd="docker push myregistry.azurecr.io/staicweb:$imageTag"
    echo $pushcmd
    eval "$pushcmd"

    # Change to the artifcat working directory
    cd _static-web-app
    ls

    # Prepare to deploy
    # Create kubeconfig file to run kubectl command. It is stored in a pipeline variable CONFIG
    echo $CONFIG | base64 --decode >> config

    kubectl create configmap css-file --from-file=site.css -n devops-demo --dry-run=client -o yaml | kubectl apply --kubeconfig=config -f -

    # replace the image tag in the manifest file
    sed -i "s/{tag}/$imageTag/g" deploy.yaml

    kubectl apply -f deploy.yaml -n devops-demo  --kubeconfig=config

```

If deploying to app service, replace teh deployment step in the script with cli command for [app service](https://docs.microsoft.com/en-us/cli/azure/webapp/config/container?view=azure-cli-latest#az-webapp-config-container-set) with [storage mount](https://docs.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage?pivots=container-linux)

Whenever the site.css is updated, the release pipeline is triggered and deploys to the target without the need to rebuilding the image.

The deployment can be extended through GitOps which will be explained in another post.
