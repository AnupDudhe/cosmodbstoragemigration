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
###  This function runs daily at midnight to archive data older than three months. (IN C#)
```
using System;
using System.IO;
using System.Text.Json;
using System.Threading.Tasks;
using Azure.Storage.Blobs;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

public class ArchiveOldData
{
    private readonly ILogger _logger;

    public ArchiveOldData(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<ArchiveOldData>();
    }

    [Function("ArchiveOldData")]
    public async Task Run([TimerTrigger("0 0 0 * * *")] TimerInfo myTimer)
    {
        _logger.LogInformation($"ArchiveOldData function executed at: {DateTime.Now}");

        // TODO: Connect to Cosmos DB and retrieve records older than 3 months
        // Example: var oldRecords = await cosmosDbService.GetOldRecordsAsync();

        // TODO: Serialize records to JSON
        // Example: var jsonData = JsonSerializer.Serialize(oldRecords);

        // TODO: Upload JSON data to Azure Blob Storage
        // Example:
        // var blobServiceClient = new BlobServiceClient("<connection-string>");
        // var containerClient = blobServiceClient.GetBlobContainerClient("archived-data");
        // var blobClient = containerClient.GetBlobClient($"archive-{DateTime.UtcNow:yyyy-MM-dd}.json");
        // await blobClient.UploadAsync(new MemoryStream(Encoding.UTF8.GetBytes(jsonData)), overwrite: true);
    }
}
# Replace the TODO sections with your logic to connect to Cosmos DB, retrieve old records, serialize them, and upload to Blob Storage.​
#Ensure that the Blob Storage connection string and container name are correctly specified.
###  This function runs daily at midnight to archive data older than three months. (IN Python)
```
import datetime
import logging
import os
import json
from azure.storage.blob import BlobServiceClient
from azure.cosmos import CosmosClient
import azure.functions as func

def main(mytimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.datetime.utcnow().replace(tzinfo=datetime.timezone.utc).isoformat()
    logging.info(f'Python timer trigger function ran at {utc_timestamp}')

    # Retrieve environment variables
    cosmos_endpoint = os.environ['COSMOS_ENDPOINT']
    cosmos_key = os.environ['COSMOS_KEY']
    cosmos_database = os.environ['COSMOS_DATABASE']
    cosmos_container = os.environ['COSMOS_CONTAINER']
    blob_connection_string = os.environ['AZURE_STORAGE_CONNECTION_STRING']
    blob_container_name = os.environ['BLOB_CONTAINER_NAME']

    # Initialize Cosmos DB client
    cosmos_client = CosmosClient(cosmos_endpoint, cosmos_key)
    database = cosmos_client.get_database_client(cosmos_database)
    container = database.get_container_client(cosmos_container)

    # Calculate the cutoff date (e.g., 3 months ago)
    cutoff_date = datetime.datetime.utcnow() - datetime.timedelta(days=90)

    # Query for records older than the cutoff date
    query = f"SELECT * FROM c WHERE c.timestamp < '{cutoff_date.isoformat()}'"
    old_records = list(container.query_items(query=query, enable_cross_partition_query=True))

    if not old_records:
        logging.info("No old records found for archiving.")
        return

    # Serialize records to JSON
    json_data = json.dumps(old_records, default=str)

    # Initialize Blob Storage client
    blob_service_client = BlobServiceClient.from_connection_string(blob_connection_string)
    blob_container_client = blob_service_client.get_container_client(blob_container_name)

    # Create a unique blob name
    blob_name = f"archive-{datetime.datetime.utcnow().strftime('%Y-%m-%d')}.json"

    # Upload JSON data to Blob Storage
    blob_client = blob_container_client.get_blob_client(blob_name)
    blob_client.upload_blob(json_data, overwrite=True)

    logging.info(f"Archived {len(old_records)} records to blob '{blob_name}' in container '{blob_container_name}'.")
```

```
### Fetch archived data from Azure Blob Storage when requested by a client. 
### this configuration can be really crucial in retrieving 3months older data when client requests and giving the data access in few seconds
```
#This function retrieves archived data from Azure Blob Storage based on the provided date.

using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;
using Azure.Storage.Blobs;
using Microsoft.Azure.Cosmos;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;

public class RestoreRequest
{
    [JsonPropertyName("blobName")]
    public string BlobName { get; set; }
}

public class RestoreArchivedData
{
    private readonly ILogger _logger;

    public RestoreArchivedData(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<RestoreArchivedData>();
    }

    [Function("RestoreArchivedData")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "restore")] HttpRequestData req,
        FunctionContext executionContext)
    {
        _logger.LogInformation("Processing request to restore archived data.");

        // Parse request body to get blob name
        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        RestoreRequest data = JsonSerializer.Deserialize<RestoreRequest>(requestBody);
        string blobName = data?.BlobName;

        if (string.IsNullOrEmpty(blobName))
        {
            var badResponse = req.CreateResponse(HttpStatusCode.BadRequest);
            await badResponse.WriteStringAsync("Blob name is required.");
            return badResponse;
        }

        // Access the blob
        string blobConnectionString = Environment.GetEnvironmentVariable("AzureWebJobsStorage");
        BlobServiceClient blobServiceClient = new BlobServiceClient(blobConnectionString);
        BlobContainerClient containerClient = blobServiceClient.GetBlobContainerClient("archived-data");
        BlobClient blobClient = containerClient.GetBlobClient(blobName);

        if (!await blobClient.ExistsAsync())
        {
            var notFoundResponse = req.CreateResponse(HttpStatusCode.NotFound);
            await notFoundResponse.WriteStringAsync("Blob not found.");
            return notFoundResponse;
        }

        var downloadInfo = await blobClient.DownloadAsync();
        using var reader = new StreamReader(downloadInfo.Value.Content);
        string jsonData = await reader.ReadToEndAsync();

        // Deserialize JSON data
        List<YourDataModel> records;
        try
        {
            records = JsonSerializer.Deserialize<List<YourDataModel>>(jsonData);
        }
        catch (JsonException ex)
        {
            _logger.LogError($"JSON deserialization error: {ex.Message}");
            var errorResponse = req.CreateResponse(HttpStatusCode.BadRequest);
            await errorResponse.WriteStringAsync("Invalid JSON format in blob.");
            return errorResponse;
        }

        // Insert records into Cosmos DB
        string cosmosConnectionString = Environment.GetEnvironmentVariable("CosmosDbConnectionString");
        string databaseName = Environment.GetEnvironmentVariable("CosmosDbDatabaseName");
        string containerName = Environment.GetEnvironmentVariable("CosmosDbContainerName");

        CosmosClient cosmosClient = new CosmosClient(cosmosConnectionString);
        Container container = cosmosClient.GetContainer(databaseName, containerName);

        foreach (var record in records)
        {
            try
            {
                await container.UpsertItemAsync(record);
            }
            catch (CosmosException ex)
            {
                _logger.LogError($"Error inserting record with ID {record.Id}: {ex.Message}");
                // Optionally, handle individual record failures
            }
        }

        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteStringAsync("Data restored successfully.");
        return response;
    }
}


```
### To invoke this function, send an HTTP POST request to the function's endpoint with a JSON body specifying the blob name:
```
{
  "blobName": "archive-2025-01-01.json"
}
```
now these two functions will ensure your timely older record is stored in blob in schedula based and your cosmos storage is reduced and the other function will ensure that whenever client requets older data of cosmosdb


### Core Azure Services required 
Azure Cosmos DB 
A globally distributed, multi-model NoSQL database service.​
Stores your primary, operational data.​
Supports triggers and change feed for real-time data processing

Azure Blob Storage
Object storage solution for the cloud.​
Used to archive data from Cosmos DB.​
Supports different access tiers: Hot, Cool, and Archive.

Azure Functions
Serverless compute service to run event-driven code.
Implements:​
Scheduled Function: Periodically moves data older than a specified duration from Cosmos DB to Blob Storage.
HTTP Triggered Function: Restores archived data from Blob Storage back to Cosmos DB upon client request.

### Workflow Overview
Data Archiving
A scheduled Azure Function queries Cosmos DB for records older than a specified duration (e.g., 3 months).​
These records are serialized and stored in Azure Blob Storage under the Cool or Archive tier, depending on access requirements.​

Data Restoration
Upon client request, an HTTP-triggered Azure Function retrieves the specified blob from Azure Blob Storage.​
The function deserializes the data and inserts it back into Cosmos DB, making it immediately accessible to the client.
