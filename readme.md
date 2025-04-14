### login your azure credentials
```
az login  #enter your azure credentials ensure you have azure cli installed on your shell.
```
### creation of resourcegroupifrequired
```
az group create \
  --name insertyourresourcegroupname \
  --location eastus
```
### Create a Storage Account with Cool Access Tier
```
az storage account create \
  --name mystorageaccount \
  --resource-group MyResourceGroup \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Cool
```
### Create a Blob Container (Optional)
```
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount \
  --auth-mode login
```






### Set Up Azure Function & Create a Timer-Triggered Azure Function:
### Connect to Cosmos DB: Use the Cosmos DB SDK to connect to your database.​

### Query Data: Retrieve documents where the createdAt field is older than three months.​

### Store in Blob Storage: Serialize the retrieved documents into a file (e.g., JSON) and upload it to Azure Blob Storage.

### Select the Timer Trigger template.​
### Define the schedule using a CRON expression. For daily execution at midnight:
```
 "schedule": "0 0 0 * * *"
```

```
import datetime
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient

def main(mytimer: func.TimerRequest) -> None:
    # Initialize Cosmos DB client
    cosmos_client = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    database = cosmos_client.get_database_client(DATABASE_NAME)
    container = database.get_container_client(CONTAINER_NAME)

    # Calculate the date three months ago
    three_months_ago = datetime.datetime.utcnow() - datetime.timedelta(days=90)

    # Query for documents older than three months
    query = "SELECT * FROM c WHERE c.createdAt < @date"
    parameters = [{"name": "@date", "value": three_months_ago.isoformat()}]
    items = list(container.query_items(query=query, parameters=parameters, enable_cross_partition_query=True))

    # Serialize and upload to Blob Storage
    blob_service_client = BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
    blob_client = blob_service_client.get_blob_client(container=BLOB_CONTAINER_NAME, blob="archived_data.json")
    blob_client.upload_blob(json.dumps(items), overwrite=True)
```

