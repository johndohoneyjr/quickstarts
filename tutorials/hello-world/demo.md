# Initial Setup and variable assignments
```
az login
az extension add --name containerapp
az provider register --namespace Microsoft.App
RESOURCE_GROUP="my-container-apps"
LOCATION="canadacentral"
CONTAINERAPPS_ENVIRONMENT="my-environment"
```

# Create the Resource Group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
  
#  Create the environment 
Individual container apps are deployed to an Azure Container Apps environment.
```
az containerapp env create \
  --name $CONTAINERAPPS_ENVIRONMENT \
  --resource-group $RESOURCE_GROUP \
  --location "$LOCATION"
```
# Setup Storage Account to persist demo messages
```
STORAGE_ACCOUNT="<storage account name>"
STORAGE_ACCOUNT_CONTAINER="mycontainer"
  
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location "$LOCATION" \
  --sku Standard_RAGRS \
  --kind StorageV2
```

# Get the storage account key
```
STORAGE_ACCOUNT_KEY=`az storage account keys list --resource-group $RESOURCE_GROUP --account-name $STORAGE_ACCOUNT --query '[0].value' --out tsv`
```  
statestore.yaml for Azure Blob storage component.  
This file helps enable your Dapr app to access your state store.
```
componentType: state.azure.blobstorage
version: v1
metadata:
- name: accountName
  value: "<STORAGE_ACCOUNT>"
- name: accountKey
  secretRef: account-key
- name: containerName
  value: mycontainer
secrets:
- name: account-key
  value: "<STORAGE_ACCOUNT_KEY>"
scopes:
- nodeapp
```  
Navigate to the directory in which you stored the statestore.yaml file and run the following command 
to configure the Dapr component in the Container Apps environment
```
az containerapp env dapr-component set \
    --name $CONTAINERAPPS_ENVIRONMENT --resource-group $RESOURCE_GROUP \
    --dapr-component-name statestore \
    --yaml statestore.yaml
``  
# Deploy the WebServer:
This service (Node) runs as an app server on --target-port 3000 (the app port)
its accompanying Dapr sidecar configured with --dapr-app-id nodeapp and
--dapr-app-port 3000 for service discovery and invocation
```  
az containerapp create \
  --name nodeapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image dapriosamples/hello-k8s-node:latest \
  --target-port 3000 \
  --ingress 'external' \
  --min-replicas 1 \
  --max-replicas 1 \
  --enable-dapr \
  --dapr-app-port 3000 \
  --dapr-app-id nodeapp
 ``` 
 This command deploys pythonapp that also runs with a Dapr sidecar that is used to look up and securely 
 call the Dapr sidecar for nodeapp. As this app is headless there is no --target-port to start a server, 
 nor is there a need to enable ingress.
```
az containerapp create \
  --name pythonapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image dapriosamples/hello-k8s-python:latest \
  --min-replicas 1 \
  --max-replicas 1 \
  --enable-dapr \
  --dapr-app-id pythonapp
```

# Confirm successful state persistence
You can confirm that the services are working correctly by viewing data in your Azure Storage account.
Open the Azure portal in your browser and navigate to your storage account.
Select Containers left side menu.
Select mycontainer.
Verify that you can see the file named order in the container.
Select on the file.
Select the Edit tab.
Select the Refresh button to observe how the data automatically updates.

# View Logs

Data logged via a container app are stored in the ContainerAppConsoleLogs_CL custom table in the Log Analytics workspace. 
You can view logs through the Azure portal or with the CLI. Wait a few minutes for the analytics to arrive for the first 
time before you are able to query the logged data.
```
LOG_ANALYTICS_WORKSPACE_CLIENT_ID=`az containerapp env show --name $CONTAINERAPPS_ENVIRONMENT --resource-group $RESOURCE_GROUP --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId --out tsv`

az monitor log-analytics query \
  --workspace $LOG_ANALYTICS_WORKSPACE_CLIENT_ID \
  --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == 'nodeapp' and (Log_s contains 'persisted' or Log_s contains 'order') | project ContainerAppName_s, Log_s, TimeGenerated | take 5" \
  --out table
```  
# Clean Up Demo
```
az group delete \
    --resource-group $RESOURCE_GROUP
```  
