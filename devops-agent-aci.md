# Hosting DevOps Agent on ACI

Self-hosted devops agents are required to access resources and perform deployments within a virtual network. Below is an outline of the steps to deploy the agent on Azure Container Instances:

1. Create a [Docker image](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux) for the agent 
2. [Push](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli) the image to an image repository

    ```
        docker push myregistry.azurecr.io/dockeragent:latest
    ```
3. Create a new Agent Pool in *Organization Settings -> Agent Pool*
4. Create a PAT token from User *Settings -> Personal Access Tokens* with the scope **Agent Pools (read, manage)**
5. [Deploy](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-environment-variables) the agent to ACI
    ```
    az container create \
        --resource-group myResourceGroup \
        --name devopsagent \
        --image myregistry.azurecr.io/dockeragent:latest \
        --restart-policy OnFailure \
        --environment-variables 'AZP_URL'='<https://dev.azure.com/orgname>'  'AZP_TOKEN'='<PAT_token>' 'AZP_AGENT_NAME'='<agent_name>' 'AZP_POOL'='<pool_name>'
    ```
Note: Required tools can be installed in the image so that a customized image is built to the need.