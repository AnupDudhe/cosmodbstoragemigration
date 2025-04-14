### Set Up Azure Function
### Connect to Cosmos DB: Use the Cosmos DB SDK to connect to your database.​

### Query Data: Retrieve documents where the createdAt field is older than three months.​

### Store in Blob Storage: Serialize the retrieved documents into a file (e.g., JSON) and upload it to Azure Blob Storage.

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

