# Azure Functions and KEDA

This is a regular Azure Function C# project triggered by Azure Service Bus topic.

As Kubernetes (AKS) will be our main orchestrator, I've opted to deploy Azure Functions to AKS and leverage the KEDA project.

The primary services of the platform are event driven, KEDA and Azure Functions make it really easy to build such scenarios so you can focus on building the actual services rather than worrying bout the pluming.

>NOTE: Read more about [KEDA and Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-kubernetes-keda) here. Also check out this [1.0 Release](https://cloudblogs.microsoft.com/opensource/2019/11/19/keda-1-0-release-kubernetes-based-event-driven-autoscaling/) by Mr. Serverless (Jeff Hollan).

Check out also these samples [kedacore/samples](https://github.com/kedacore/samples)

## Quick Tips

### Azure Functions Core Tools

As part for the prerequisites, you should have installed Azure Functions Core Tools. Please refer back to the setup documentation for further details.

### Creating new Azure Function in VS Code

You can leverage the Azure Functions VS Code extension to easily create new function.

You can also use Azure Functions Core tools to do as well through ```func new```

### Custom NuGet Source

As the project uses a custom feed to consume tailored and none-public packages, you can find a [nuget.config](nuget.config) file adding Azure DevOps Artifacts as source.

You can use the following command to add custom package if you are using VS Code:

```shell

dotnet add package CoreLib -s https://ORGANIZATION.pkgs.visualstudio.com/PROJECT/_packaging/Mo.Packages/nuget/v3/index.json

```

In Visual Studio, the experience is a bit easier. Just go to the settings and add a new custom NuGet source which will then allow you to use the normal **Manage NuGet Packages** project right click action super simple. Just change the search in option from the drop down list in the top right.

### Adding Docker Support

To generate a docker file on your existing Azure Function project, just run the following command (make sure you are in the Function project root directory):

```bash

func init . --docker-only

```

>NOTE: Check the generated Dockerfile and update the path ```/src/dotnet-function-app``` to your relevant function app folder.

It is worth mentioning that the generated docker is doing a multi-stage container building. This means it uses SDK container (fat container) to build your source code, then build another runtime-only container (leaner). This allows you to have platform-independent build (you don't need the machine building the container to have .NET Core SDK installed for example)

### Update local.settings.json

This particular function requires various settings to be present at runtime (like Azure Storage and service bus connections). Updating the local.settings.json will allow the automatically generated Kubernetes deployment to have the needed Kubernetes secrets.

This is a snapshot that you can update and include in your local.settings.json:

```json

{
  "IsEncrypted": false,
  "serviceBus": {
    "prefetchCount": 100,
    "messageHandlerOptions": {
        "autoComplete": true,
        "maxConcurrentCalls": 32,
        "maxAutoRenewDuration": "00:55:00"
    }
  },
  "Values": {
    "AzureWebJobsStorage": "REPLACE",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",
    "serviceBusConnection": "REPLACE"
  }
}


```

### Deploy KEDA Runtime to Kubernetes

In order to leverage KEDA in any Kubernetes cluster, you need to deploy KEDA components first.

Azure Functions Core Tools make it easy using the following command:

```bash

func kubernetes install --namespace keda

```

>NOTE: Make sure your ```kubectl``` context is set to the target kubernetes cluster

Review that installation was successful and in running state:

```bash

kubectl get po -n keda

# You should get something like this:
# NAME                                                       READY   STATUS    RESTARTS   AGE
# keda-59748d7c48-l226t                                      1/1     Running   0          26s
# osiris-osiris-edge-activator-6f6c7db4dd-r4xzl              1/1     Running   0          21s
# osiris-osiris-edge-endpoints-controller-74f5cc99f8-h64qg   1/1     Running   0          21s
# osiris-osiris-edge-endpoints-hijacker-5b4776fb9b-8b6s9     1/1     Running   0          20s
# osiris-osiris-edge-proxy-injector-769cbdfcc7-5l8j6         1/1     Running   0          20s
# osiris-osiris-edge-zeroscaler-754f77b757-cqz88             1/1     Running   0          19s

# KEDA create also a new (scaledobjects.keda.k8s.io) CRS which will be used in kubernetes deployments.
kubectl get customresourcedefinition

```

>NOTE: If you deployed Azure Firewall in a secure AKS deployment, make sure that a rule to allow osiris images to be pulled from osiris.azurecr.io/osiris:6b69328 or the pods will fail to start

If you faced issues in deploying KEDA, you can remove the deployment and consult the [KEDA documentations](https://keda.sh/)

```bash

func kubernetes remove --namespace keda

```

### Generating Kubernetes Deployment Manifest

Deploying to Kubernetes is also super simple, and Azure Functions Core Tools help by generating (or executing) the deployment yaml file.

```bash

func kubernetes deploy --name cognitive-orchestrator --registry $CONTAINER_REGISTRY_NAME.azurecr.io/services --dotnet --dry-run > deploy-updated.yaml

```

Check out the generated deployment file and update it accordingly.

>NOTE: If you are pushing to private Azure Container Registry, please make sure you are authenticated correctly. Refer to [prerequisites guidelines](../../../guide/02-prerequisites/README.md) for more information.

If you need to leverage AKS Virtual Nodes capability (for a serverless infinite scale), you can amend the tolerations to allow deployment to both cluster nodes and virtual nodes:

```yaml

tolerations:
  - operator: Exists

```

>NOTE: To use AKS Virtual Nodes, you need first to enable it on your AKS cluster. Check out the [documentation here using Azure CLI](https://docs.microsoft.com/en-us/azure/aks/virtual-nodes-cli)

Another important note when using KEDA with Azure Service Bus, you need to have a connection string that scoped at the level of the topic (not at the namespace level). That is why I added a special service bus connection SAS into a separate variable under the secrets deployment ``` KEDA_SERVICE_BUS_CONNECTION ```.

>NOTE: You can check the current enhancement issue mentioned above on [GitHub](https://github.com/kedacore/keda/issues/215)

>NOTE: For simplicity, the function app uses only one connection to Service Bus to both receive from one topic and send to another. It can easily done through update the configuration and code to use 2 different connection string for each Service Bus Topics.

One you are satisfied with the generated deployment file, copy the file to the deployment folder. This is to allow Azure DevOps to copy it out so it can be used in the release pipeline.

#### Diagnose KEDA deployment

If something is not going right, you can check directly KEDA logs:

```bash

kubectl logs $REPLACE_WITH_KEDA_POD_NAME -n keda

```

#### Sample Deployment File

Note that all caps values are to be replace with values related to your deployment.

Also note that this deployment files adds ```tolerations``` to instruct KEDA to leverage AKS Virtual Nodes.

You can find the deployment file used in the DevOps process here [Deployment/k8s-deployment.yaml](Deployment/k8s-deployment.yaml)

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: cognitive-orchestrator
  namespace: crowd-analytics
data:
  AzureWebJobsStorage: #{funcStorage}#
  FUNCTIONS_WORKER_RUNTIME: #{funcRuntime}#
  APPINSIGHTS_INSTRUMENTATIONKEY: #{appInsightsKey}#
  serviceBusConnection: #{serviceBusConnection}#
  KEDA_SERVICE_BUS_CONNECTION: #{kedaServiceBusConnection}#
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cognitive-orchestrator
  namespace: crowd-analytics
  labels:
    app: cognitive-orchestrator
spec:
  selector:
    matchLabels:
      app: cognitive-orchestrator
  template:
    metadata:
      labels:
        app: cognitive-orchestrator
    spec:
      containers:
      - name: cognitive-orchestrator
        image: #{acrName}#/crowdanalytics/cognitive-orchestrator:#{Build.BuildId}#
        env:
        - name: AzureFunctionsJobHost__functions__0
          value: CognitiveOrchestrator
        envFrom:
        - secretRef:
            name: cognitive-orchestrator
---
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: cognitive-orchestrator
  namespace: crowd-analytics
  labels:
    deploymentName: cognitive-orchestrator
spec:
  scaleTargetRef:
    deploymentName: cognitive-orchestrator
  pollingInterval: 30  # Optional. Default: 30 seconds
  cooldownPeriod:  300 # Optional. Default: 300 seconds
  minReplicaCount: 0   # Optional. Default: 0
  maxReplicaCount: 100 # Optional. Default: 100
  triggers:
  - type: azure-servicebus
    metadata:
      type: serviceBusTrigger
      connection: KEDA_SERVICE_BUS_CONNECTION
      topicName: cognitive-request
      subscriptionName: cognitive-orchestrator
      name: request
---

```