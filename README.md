---
page_type: sample
languages:
- javascript
products:
- azure
description: "This sample implements a function triggered by Azure Blob Storage to resize an image in Node.js."
urlFragment: storage-blob-resize-function-node
---

# Azure Storage Blob Trigger Image Resize Function in Node.js

This sample implements a function triggered by Azure Blob Storage to resize an image in Node.js. Once the image is resized, the thumbnail image is uploaded back to blob storage.

The key aspects of this sample are in the function bindings and implementation.

## Function bindings
In order to interface with image data, you need to configure the function to process binary data.

The following code sets the `datatype` parameter to `binary` in the `function.json` file.

```javascript
{
  "disabled": false,
  "bindings": [
    {
      "type": "eventGridTrigger",
      "name": "myEvent",
      "direction": "in"
    },
    {
      "name": "myBlob",
      "type": "blob",
      "direction": "in",
      "path": "{data.url}",
      "connection": "AZURE_STORAGE_CONNECTION_STRING",
      "datatype": "binary"
    }
  ]
}
```

## Function implementation

The sample uses [Jimp](https://github.com/oliver-moran/jimp) to resize an incoming buffer to a thumbnail. The buffer is then converted to a stream (as required by [createBlockBlobFromStream](https://docs.microsoft.com/en-us/javascript/api/azure-storage/blobservice?view=azure-node-latest#createblockblobfromstream-container--blob---stream---streamlength--options--callback-)) and uploaded to Azure Storage.


```javascript
const Jimp = require('jimp');
const stream = require('stream');
const { BlockBlobClient } = require("@azure/storage-blob");

const ONE_MEGABYTE = 1024 * 1024;
const uploadOptions = { bufferSize: 4 * ONE_MEGABYTE, maxBuffers: 20 };

const containerName = process.env.BLOB_CONTAINER_NAME;
const connectionString = process.env.AZURE_STORAGE_CONNECTION_STRING;
const blobName = process.env.OUT_BLOB_NAME;

module.exports = async function (context, myEvent, inputBlob){
    const widthInPixels = 100;
    Jimp.read(inputBlob).then((thumbnail) => {
        thumbnail.resize(widthInPixels, Jimp.AUTO);
        thumbnail.getBuffer(Jimp.MIME_PNG, async (err, buffer) => {
          
            const readStream = stream.PassThrough();
            readStream.end(buffer);
            const blobClient = new BlockBlobClient(connectionString, containerName, blobName);
            
            try {
                await blobClient.uploadStream(readStream,
                    uploadOptions.bufferSize,
                    uploadOptions.maxBuffers,
                    { blobHTTPHeaders: { blobContentType: "image/jpeg" } });
            } catch (err) {
                context.log(err.message);
            }
        });
    });
};
```

You can use the [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/) to view blob containers and verify the resize is successful.

## Notes env implementation

#source link https://learn.microsoft.com/en-us/azure/event-grid/storage-upload-process-images?tabs=dotnet%2Cazure-powershell

#Create a storage account in the resource group
$blobStorageAccount="imagesiasx-blobStAcc"
$resourceGroupName="preditiva.prod"
$location="brazilsouth"
$container1Name="imgsuploaded"
$container2Name="imgsresized"

New-AzStorageAccount -ResourceGroupName $resourceGroupName -Name $blobStorageAccount -SkuName Standard_LRS -Location $location -Kind StorageV2 -AccessTier Hot

#Get the storage account key then use this key to create two containers 
$blobStorageAccountKey = ((Get-AzStorageAccountKey -ResourceGroupName $myResourceGroup -Name $blobStorageAccount)| Where-Object {$_.KeyName -eq "key1"}).Value
$blobStorageContext = New-AzStorageContext -StorageAccountName $blobStorageAccount -StorageAccountKey $blobStorageAccountKey

New-AzStorageContainer -Name $container1Name -Context $blobStorageContext
New-AzStorageContainer -Name $container2Name -Permission Container -Context $blobStorageContext


#Automate resizing uploaded images using Event Grid
$resourceGroupName="preditiva.prod"
$location="brazilsouth"
$blobStorageAccount="imagesiasx"
#Create an Azure Storage account - (Not necessary)
#New-AzStorageAccount -ResourceGroupName $resourceGroupName -AccountName $blobStorageAccount -Location $location -SkuName Standard_LRS -Kind StorageV2

#Create a function app
$functionapp="imagesiasx-funcApp"  
New-AzFunctionApp -Location $location -Name $functionapp -ResourceGroupName $resourceGroupName -Runtime PowerShell -StorageAccountName $blobStorageAccount

#Deploy the function code
az functionapp deployment source config --name $functionapp --resource-group $resourceGroupName --branch master --manual-integration --repo-url https://github.com/csxdigital/storage-blob-resize-function-node
             

#Configure the function app for Node.js v10 SDK
$blobStorageAccountKey=$(az storage account keys list -g $resourceGroupName -n $blobStorageAccount --query [0].value --output tsv)
$storageConnectionString=$(az storage account show-connection-string --resource-group $resourceGroupName --name $blobStorageAccount --query connectionString --output tsv)
az functionapp config appsettings set --name $functionapp --resource-group $resourceGroupName --settings FUNCTIONS_EXTENSION_VERSION=~2 BLOB_CONTAINER_NAME=$container2Name AZURE_STORAGE_ACCOUNT_NAME=$blobStorageAccount AZURE_STORAGE_ACCOUNT_ACCESS_KEY=$blobStorageAccountKey AZURE_STORAGE_CONNECTION_STRING=$storageConnectionString FUNCTIONS_WORKER_RUNTIME=node WEBSITE_NODE_DEFAULT_VERSION=~10



#For the error "The subscription is not registered to use namespace 'Microsoft.EventGrid'"
Register-AzureRmResourceProvider -ProviderNamespace Microsoft.EventGrid



# CONFIGURAÇÃO DO SERVIÇO PARA HOSPEDAR IMAGENS - AZURE BLOB STORAGE
VITE_CONTAINER_NAME='imgsresized' or 'imgsuploaded'
VITE_STORAGE_SAS_TOKEN='?sv=2021-06-08&ss=bfqt&srt=co&sp=rwdlacupitfx&se=3000-01-24T04:27:38Z&st=2023-01-23T20:27:38Z&spr=https&sig=J04jORMdTymGtZfIP2hgKSiwZFTMCVBhXQuAFjmROE4%3D'
VITE_STORAGE_RESOURCE_NAME='imagesiasx'