# My Local Dev setup for Web Application Development

## Introduction

When developing web applications for Azure, there are times when I want to have a completely local environment to develop and test in.

In order to accomplish that I need to have the following things available:

- SQL Server
- Azure Blobs
- Azure Tables
- Azube Queues

Also, since I develop on a Macbook Pro, I dont have access to the normal tooling one would find under windows.

![My Setup](images/Application%20Overview.png)

So, my environment is as follows:

- SQLServer running in docker
- Azurite also running in docker
- VScode for front end work
- Angular 13 (generally I try to keep current)
- Angular Material
- Dotnet Core 6 (or whatever the latest is)
- Visual Studio for MAC for debugging
- Rider for debugging also
- Within VSCode I have extensions for lint, markdown and a few more things to help with code quality
- Azure Storage Explorer (Looking at what I stick into blobs, tables and queues)
- Azure Data Studio (SQL work)

## Setting up sql server and azurite in docker

- Download the above tools and also docker desktop for mac
- Create a local storage folder for SQL Data and Azurite (this is so you dont lose data when shutting down the docker instance)
- Create a yaml file for docker:

``` yaml
Version: "3.9"

services:
  sqlserver:
    image: mcr.microsoft.com/azure-sql-edge
    container_name: sqlserver
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=P@55w0rd
    ports:
      - "1433:1433"
    volumes:
      - ~/DockerData/sqlserver:/var/opt/mssql/data


  azurite:
    image: mcr.microsoft.com/azure-storage/azurite
    container_name: azurite
    command: "azurite --loose --blobHost 0.0.0.0 --blobPort 10000 --queueHost 0.0.0.0 --queuePort 10001 --tableHost 0.0.0.0 --tablePort 10002 --location /workspace --debug /workspace/debug.log"
    ports:
      - 10010:10000
      - 10011:10001
      - 10012:10002
    volumes:
      - ~/DockerData/azurite:/workspace
```

The volumes tags are for mapping directories in the docker container to local directories on your mac.

In my case, I wanted to keep SQL volumes at ~/DockerData/sqlserver and blobs, queues, and tables at ~/DockerData/azurite

You'll need both Azure Storage Explorer  and Azure Data Studio to view the results of your work.

When I'm developing, I simply go to the directory with the docker-compose.yml in it and do a: docker compose up from the terminal (iterm)

## Sample Code to handle blobs

Interface definition

```csharp
    public interface IBlobRepository
    {
        void CreateBlobContainer(string containerId);
        Task<byte[]> DownloadBlob(string containerId, string blobId);
        void UploadBlob(string containerId, string blobId, byte[] content, string contentType);
    }
```

Implementation

```csharp
    public class BlobRepository : IBlobRepository
    {
        private BlobServiceClient _blobServiceClient;

        public BlobRepository(string connectionString)
        {
            _blobServiceClient = new BlobServiceClient(connectionString);
        }

        // Create a container if it doesnt exist
        public void CreateBlobContainer(string containerId)
        {
            BlobContainerClient containerClient = _blobServiceClient.GetBlobContainerClient(containerId);
            containerClient.CreateIfNotExists();
        }


        public async Task<byte[]> DownloadBlob(string containerId, string blobId)
        {
            var blobClient = _blobServiceClient.GetBlobContainerClient(containerId);
            var result = await blobClient.GetBlobClient(blobId).DownloadContentAsync();
            var bytes = result.Value.Content.ToArray();
            return bytes;
        }

        public string ReadBlob(string containerId, string blobId)
        {

            var blobClient = _blobServiceClient.GetBlobContainerClient(containerId);
            var result = blobClient.GetBlobClient(blobId).DownloadContentAsync().Result;
            byte[] bytes = result.Value.Content.ToArray();
            return Encoding.Default.GetString(bytes);
        }


        public void UploadBlob(string containerId, string blobId, byte[] content, string contentType)
        {
            var blobClient = _blobServiceClient.GetBlobContainerClient(containerId);

            using (var ms = new MemoryStream(content))
            {
                var existingClient = _blobServiceClient.GetBlobContainerClient(containerId).GetBlobClient(blobId);
                Azure.Response<BlobContentInfo> response = existingClient.Upload(ms, true);
            }
        }
    }
```

Note, when using the repository class, make sure to not keep creating a new instance of the repository inside a loop or even to use a using block on the connection. If you keep doing this, you'll eventually run into a port starvation issue. This is because when closing a connection, the network connection goes into a TIME_WAIT condition and doesnt actually free the port until it falls out of that. I found this out in a production scenario where things appeared to have ground to a halt.

One thing that I found incredibly useful was to install Parallels for Mac (ARM) and Windows 11 for ARM and download Visual Studio 2022 and SQL Server Management Studio.

In other to connect to SQL Server or other services running in the docker instances on the MAC you just need to reference them via the public IP address of the MAC. This can be obtained by doing:

``` bash
(base) ➜  docker ifconfig -a | grep 192
inet 192.168.1.249 netmask 0xffffff00 broadcast 192.168.1.255
(base) ➜  docker
```

which in my case resulted in 192.168.1.249

So, when bringing up SSMS on the windows ARM side, I just need to provide the following:

![Login](images/SQL%20Server%20Login.png)

use the password from the yaml file (P@55w0rd)

You can ofcourse use Azure Data Studio to talk to SQL Server, but it's not as nice as SSMS in my view.

## What's next?

Next up is a series of quick starts on how to build various applications using this tech stack.

In some cases, I might use other databases (relational or nosql) and in those cases, will introduce the YAML files for those as I use them.
