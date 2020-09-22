# AutoScale DevOps Agent using HPA

After hosting the DevOps agent on a Kubernetes Cluster, it may be need to auto-scale the number of agents depending on the number of queued builds.

To auto-scale the agent, this post explores using Kubernetes  Horizontal Pod Autoscaler (HPA) with an external metric exposed by the Azure DevOps REST API.

Note: The pre-requsite to this post is to have an Image for the DevOps agent as detailed [here](ci-containers.md). 

*Refer:*
- [Containerizing the self-hosted agents](ci-containers.md)
- [The Containerized agents can be deployed to ACI or AKS](devops-agent-aci.md)
- Auto-Scaling the agents on AKS

The components to achieve this are
- Metrics Provider
- Metrics Aggregator
- HPA Controller
- Agent Deployment

### Metrics Provider
The external metric to scale the agents will be the number of builds in a notStarted Status. The Azure DevOps Services [REST API](https://docs.microsoft.com/en-us/rest/api/azure/devops/build/builds/list?view=azure-devops-rest-6.0#buildstatus) provides the List API with a filter.

For ex. https://dev.azure.com/{organization}/{project}/_apis/build/builds?api-version=6.1-preview.6&statusFilter=notStarted

### Metrics Aggregator
The aggregator service is responsible to query the DevOps REST API and format the metric as required by Kubernetes
 - Uses a PAT token with access to list builds
 - REST API call to the devops services
 - Format the response in a kubernetes consumable format
 - Expose the API on an endpoint the APIService object expects

The format of each metric should be as below
```yml
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Service",
        "namespace": "default",
        "name": "devops-proxy",
        "apiVersion": "/v1beta1"
      },
      "metricName": "count",
      "timestamp": "2018-01-10T16:49:07Z",
      "value": "10"
    }
  ]
}
```

The endpoint that will be consumed by Kubernetes is as */apis/custom.metrics.k8s.io/v1beta1/namespaces/{namespace}/services/{serviceName}/{metricName}*

### HPA Controller
The [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) fetches the metrics from the aggregator service and decides to scale the target deployment as required.
 - APIService object to register the external API
 - HPA object to define the target, metric and the Scale

```yml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  insecureSkipTLSVerify: true
  group: custom.metrics.k8s.io
  groupPriorityMinimum: 1000
  versionPriority: 5
  service:
    name: devops-proxy
    namespace: default
  version: v1beta1
```

The HPA object for reference

```yml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-autoscaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {agentDeploymentName}
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      target:
        kind: Service
        name: devops-proxy
      metricName: count
      targetValue: 1
```

### Agent Deployment

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-agent
spec:
  selector:
    matchLabels:
      run: devops-agent
  replicas: 1
  template:
    metadata:
      labels:
        run: devops-agent
    spec:
      containers:
      - name: devops-agent
        image: myregistry.azurecr.io/dockeragent:latest
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```