# Azure Log Analytics Log Rotation

This repository contains instructions and code to automate the rotation of Azure Log Analytics logs. Logs are archived into a `.gz` file every 90 days.

## Prerequisites

- Azure Subscription
- Azure Log Analytics Workspace
- Azure Storage Account
- Azure Function App

## Steps to Set Up Log Rotation

### 1. Create a Storage Account

1. Go to the [Azure Portal](https://portal.azure.com).
2. Create a new Storage Account.
3. Inside the Storage Account, create a container to store the logs.

### 2. Create a Log Analytics Workspace

1. Ensure you have a Log Analytics Workspace set up in Azure.
2. Note down the Workspace ID for later use.

### 3. Automate Log Export

Use Azure Logic Apps or Azure Functions to periodically export logs from Log Analytics to your storage account.

### 4. Implement Log Rotation

Create an Azure Function to handle log rotation every 90 days.

#### Example Code for Azure Function

1. **Azure Function to Rotate Logs:**

   ```python
   import os
   import gzip
   from datetime import datetime, timedelta
   from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient

   # Azure Storage account settings
   storage_connection_string = 'your_storage_connection_string'
   container_name = 'your_container_name'
   log_blob_prefix = 'logs'

   # Function to rotate logs
   def rotate_logs():
       blob_service_client = BlobServiceClient.from_connection_string(storage_connection_string)
       container_client = blob_service_client.get_container_client(container_name)

       # Get the current date and calculate the new filename
       current_date = datetime.utcnow()
       new_blob_name = f"{log_blob_prefix}_{current_date.strftime('%Y%m%d%H%M%S')}.gz"
       
       # List all blobs in the container
       blobs = container_client.list_blobs(name_starts_with=log_blob_prefix)

       # Find the current log blob
       current_blob_name = None
       for blob in blobs:
           if blob.name.startswith(log_blob_prefix) and not blob.name.endswith('.gz'):
               current_blob_name = blob.name
               break
       
       if current_blob_name:
           # Rename the current log blob by copying it to the new blob
           source_blob = f"https://{container_client.account_name}.blob.core.windows.net/{container_name}/{current_blob_name}"
           new_blob_client = container_client.get_blob_client(new_blob_name)
           new_blob_client.start_copy_from_url(source_blob)
           
           # Wait for the copy to complete
           props = new_blob_client.get_blob_properties()
           while props.copy.status == "pending":
               props = new_blob_client.get_blob_properties()

           # Delete the original blob after the copy completes
           container_client.delete_blob(current_blob_name)

       # Create a new current log blob
       new_current_blob_name = f"{log_blob_prefix}_current.log"
       current_blob_client = container_client.get_blob_client(new_current_blob_name)
       current_blob_client.upload_blob("", overwrite=True)

   # Main function to execute
   def main(mytimer: func.TimerRequest) -> None:
       rotate_logs()
