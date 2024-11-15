---
title: Building an AI-Powered Application on Azure with Terraform
date: 2024-11-14
categories: [Terraform, Azure]
tags: [Terraform, Azure]
author: Lukasz Halicki
---

## Introduction

Learn how to deploy an AI-powered application on Azure using Terraform. This blog post covers the main components involved, how they interact, and how to configure them using Terraform code to build a scalable and secure solution.
In the era of artificial intelligence, integrating AI capabilities into applications has become a necessity for businesses aiming to stay competitive. Azure provides a rich ecosystem of services that make it easier to build, deploy, and scale AI-powered applications. By leveraging Terraform for infrastructure as code, you can automate the provisioning of these resources, ensuring consistency and efficiency.

In this blog post, we'll walk through how to deploy an AI-powered application on Azure using Terraform. We will cover the main components involved, how they interact, and how to configure them using Terraform code to build a scalable and secure solution.

## Data Flow

Before diving into the specific components, it’s important to understand how data flows through the architecture. This architecture automates the processing and classification of documents using Azure services, leveraging serverless functions to orchestrate the entire workflow. Here's an overview of the data flow:

1. **Document Upload**: The user uploads documents through the Azure Web App frontend. The uploaded documents are stored in an Azure Blob Storage account.
2. **Queue Message Creation**: Once a document is uploaded, an Azure Storage Queue is used to trigger the processing workflow. A message is placed in the queue to signal the start of processing.
3. **Processing with Azure Functions**:
    * An **Azure Durable Function** is triggered by the message in the queue. The durable function orchestrates the entire document classification process.
    * The **Document Intelligence** (part of Azure Cognitive Services) is used to analyze the document and extract key information (e.g., text, tables, and metadata).
4. **Document Metadata Storage**: The extracted metadata is stored in Azure Cosmos DB, providing a fast and scalable solution for storing structured information.
5. **Document Indexing**: The processed document and its metadata are indexed using **Azure Search**. This allows for efficient querying and retrieval of documents based on the extracted metadata.
6. **Result Notification**: Once the process is complete, the status of the document processing can be updated in the web application. Users can search, query, and retrieve the processed documents based on the metadata stored in Cosmos DB and indexed in Azure Search.

This flow ensures that document processing is efficient, scalable, and easy to manage, all while being orchestrated by Azure Functions and integrated with other Azure services.

## Prerequisites

Before we begin, ensure you have the following:

* **Azure Subscription**: An active Azure account with permissions to create resources.
* **Terraform Installed**: Install the latest version of Terraform on your local machine.
* **Azure CLI**: For authentication and managing Azure resources.
* **Basic Knowledge of Terraform and Azure Services**: Familiarity with Terraform syntax and Azure services like App Service, Storage Accounts, Azure Functions, Cognitive Services, Cosmos DB, and Azure Search.

## Understanding the Main Components

Our architecture consists of several Azure services, each serving a specific purpose in the AI application. Below are the components along with the corresponding Terraform code and explanations.

### 3.1 Resource Group

A **resource group** in Azure is a container that holds related resources for an Azure solution. By grouping resources together, you can manage and monitor them as a single entity. This also allows for easier application lifecycle management, such as deleting, updating, or scaling resources within the group.

**Terraform Code**:

    resource "azurerm_resource_group" "ai_usecase_rg" {
        name     = "${var.name}-rg"
        location = var.resource_group_location
    }

**Explanation**:

* **name**: The name of the resource group is generated dynamically using the variable `var.name`. This allows for a flexible naming convention, making the resource group easily identifiable.
* **location**: Specifies the Azure region (location) where the resources in this group will be deployed. Using a variable for the location (`var.resource_group_location`) makes the deployment flexible for different regions.

In this AI-powered application architecture, the resource group will contain all the necessary components, including web apps, storage accounts, functions, and other services.

### 3.2 Service Plan

The **Azure App Service Plan** defines the underlying compute resources (such as CPU, memory, and disk space) for hosting Azure Web Apps and Function Apps. The service plan is an essential part of managing cost and performance. Here, we use a Linux-based App Service Plan, which is suitable for hosting both our web application and function apps.

**Terraform Code**:

    resource "azurerm_service_plan" "service_plan" {
        name                = "${var.name}-plan"
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        location            = azurerm_resource_group.ai_usecase_rg.location
        os_type             = "Linux"
        sku_name            = "B1"
    }
    

**Explanation**:

* **name**: Like the resource group, the service plan name is generated dynamically using the variable `var.name`. This ensures consistency across related resources.
* **resource\_group\_name**: Associates the service plan with the resource group we just created, ensuring all related resources are grouped together.
* **location**: Uses the same region as the resource group for deploying the service plan.
* **os\_type**: Specifies that the underlying OS is Linux. This is suitable for hosting modern web apps and serverless functions that require Linux-based runtimes.
* **sku\_name**: Defines the pricing tier. "B1" represents a basic, low-cost tier for development or small-scale applications. For production workloads, you may choose a more robust tier (e.g., Standard or Premium).

In this architecture, the App Service Plan provides the compute resources to host the Web App and Function Apps.

### 3.3 Web App

The **Azure Web App** is the platform where the frontend of our AI application will run. This service allows you to build and host web applications in any programming language without having to manage infrastructure. Azure Web Apps automatically scale and manage the infrastructure based on demand, making it an ideal choice for modern web applications.

**Terraform Code**:

    resource "azurerm_linux_web_app" "web_app" {
        name                      = "${var.name}-web-app"
        location                  = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name       = azurerm_resource_group.ai_usecase_rg.name
        service_plan_id           = azurerm_service_plan.service_plan.id
    
        # Enable the system-assigned managed identity for the web app
        identity {
            type = "SystemAssigned"
        }
    
        # Configure the web app to use the latest
        site_config {
            always_on = false
        }
    
        depends_on = [
            azurerm_service_plan.service_plan
        ]
    }
    

**Explanation**:

* **name**: The web app is named dynamically using `var.name`, helping maintain consistent naming for related resources.
* **location** & **resource\_group\_name**: Both attributes ensure that the web app is deployed in the same region and resource group as the other components.
* **service\_plan\_id**: The web app is tied to the previously created App Service Plan, meaning it shares the same compute resources defined in that plan.
* **identity**: The web app is assigned a **System-Assigned Managed Identity**, which allows it to securely interact with other Azure services (like Storage, Cognitive Services, etc.) without needing to manage credentials manually.
* **site\_config**: The `always_on` setting is set to `false`, meaning the web app will stop when idle. This can save costs, but for production environments where the app should always be running, you can set this to `true`.

The web app serves as the user interface for document upload, providing a seamless way to interact with the backend services that will process and store the documents.

### 3.4 Storage Accounts and Containers

**Azure Storage Accounts** provide highly scalable and durable cloud storage for unstructured data like documents, images, and videos. In this architecture, the storage account will handle document uploads and store the resulting processed data.

**Terraform Code**:

    resource "azurerm_storage_account" "storage" {
        name                     = var.storage_account_name
        resource_group_name      = azurerm_resource_group.ai_usecase_rg.name
        location                 = azurerm_resource_group.ai_usecase_rg.location
        account_tier             = "Standard"
        account_replication_type = "LRS"
        account_kind             = "StorageV2"
    }
    

**Explanation**:

* **name**: Uses the variable `var.storage_account_name` to define the storage account’s name.
* **account\_tier**: Specifies the pricing tier. **Standard** is used here for a balance between cost and performance.
* **account\_replication\_type**: **LRS** (Locally Redundant Storage) ensures that the data is replicated within the same Azure region. For more critical data, you might opt for **GRS** (Geo-Redundant Storage).
* **account\_kind**: **StorageV2** provides the latest features for blob, table, and queue storage with improved performance.

The storage account is used to store unstructured data (such as documents) uploaded via the web app.

### 3.5 Managed Identities and Role Assignments

**Managed Identities** enable your web apps or functions to authenticate to Azure services (such as Storage or Cognitive Services) securely without embedding credentials in the application code. Terraform automatically creates these identities and assigns the necessary roles.

**Terraform Code**:

    resource "azurerm_role_assignment" "app_service_queue_contributor" {
        principal_id         = azurerm_linux_web_app.web_app.identity[0].principal_id
        role_definition_name = "Storage Queue Data Contributor"
        scope                = azurerm_storage_account.storage.id
    
        depends_on = [
            azurerm_linux_web_app.web_app,
            azurerm_storage_queue.queue
        ]
    }
    

**Explanation**:

* **principal\_id**: The **System-Assigned Managed Identity** of the web app is retrieved and used here to grant the necessary role.
* **role\_definition\_name**: Assigns the **Storage Queue Data Contributor** role, allowing the web app to enqueue messages to the Azure Storage Queue. This role is essential for the app to trigger background processing tasks.
* **scope**: Limits the role to the specific storage account.

The same concept is applied when assigning **Blob Data Contributor** roles, allowing the web app to interact with Blob Storage securely.

### 3.6 Azure Functions

**Azure Functions** provide a serverless compute service that allows you to run background tasks triggered by events like HTTP requests or queue messages. In this architecture, functions are used to process documents and interact with Azure Cognitive Services.

**Terraform Code**:

    resource "azurerm_linux_function_app" "functions_app" {
        for_each = var.function_app
    
        name                       = each.value.name
        resource_group_name        = azurerm_resource_group.ai_usecase_rg.name
        location                   = azurerm_resource_group.ai_usecase_rg.location
        service_plan_id            = azurerm_service_plan.service_plan.id
        storage_account_name       = azurerm_storage_account.functions_file_system_storage.name
        storage_account_access_key = azurerm_storage_account.functions_file_system_storage.primary_access_key
    
        site_config {}
    
        depends_on = [
            azurerm_service_plan.service_plan,
            azurerm_linux_web_app.web_app
        ]
    }
    

**Explanation**:

* **for\_each**: This allows for the creation of multiple function apps based on the `var.function_app` map, making it easier to scale or deploy multiple functions for different tasks.
* **service\_plan\_id**: Functions are tied to the same service plan as the web app, allowing them to share the same compute resources.
* **storage\_account\_name**: Specifies the storage account that the function app will use to store files, logs, and configurations.
* **depends\_on**: Ensures that the service plan and web app are deployed before the function app.

Functions will be used to orchestrate the document processing workflow, handling tasks like analyzing documents using Cognitive Services and updating metadata in Cosmos DB.

### 3.7 Cognitive Services

Azure **Cognitive Services** provide AI-powered capabilities like form recognition, language understanding, and image analysis. In this architecture, the **Form Recognizer** service is used to extract information from uploaded documents.

**Terraform Code**:

    resource "azurerm_cognitive_account" "document_intelligence" {
        name                = "${var.name}-document-intelligence"
        location            = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        kind                = "FormRecognizer"
        sku_name            = "S0"
    }
    

**Explanation**:

* **kind**: Specifies that we are deploying the **Form Recognizer** service.
* **sku\_name**: The **S0** tier is a standard pricing tier offering moderate processing power. For large-scale processing, you might consider higher-tier pricing plans.

This service is essential for analyzing and extracting data from documents uploaded via the web app, making the application AI-powered.

### 3.8 Cosmos DB

Azure **Cosmos DB** is a fully managed, globally distributed NoSQL database that provides high availability and low-latency access to data. In this architecture, Cosmos DB stores metadata extracted from documents.

**Terraform Code**:

    resource "azurerm_cosmosdb_account" "cosmosDB" {
        name                  = "${var.name}-cosmosdb"
        location              = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name   = azurerm_resource_group.ai_usecase_rg.name
        kind                  = "MongoDB"
        consistency_policy {
            consistency_level = "Strong"
        }
    }
    

**Explanation**:

* **kind**: Specifies that this Cosmos DB account is configured for the **MongoDB** API, making it suitable for storing JSON-based document metadata.
* **consistency\_level**: **Strong** consistency ensures that every read returns the most recent write, but this comes at the cost of higher latency. For scenarios where lower latency is more important, you could use **Session** or **Eventual** consistency.

Cosmos DB ensures that metadata from the processed documents is stored reliably and can be retrieved quickly by the application.

### 3.9 Azure Search Service

The **Azure Search Service** provides full-text search capabilities that allow users to search the processed documents based on the metadata extracted and stored in Cosmos DB. This service adds advanced search functionality to the AI application.

**Terraform Code**:

    resource "azurerm_search_service" "search" {
        name                = "${var.name}-search"
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        location            = azurerm_resource_group.ai_usecase_rg.location
        sku                 = "free"
    }
    

**Explanation**:

* **sku**: Specifies the pricing tier. In this case, the **free** tier is sufficient for development and testing purposes, but you can scale up to a paid tier for production environments.

Azure Search indexes the metadata stored in Cosmos DB and allows users to perform advanced queries, making the documents easily searchable.

## Putting It All Together

Now that we've covered all the components, it's time to see how everything ties together in the complete Terraform configuration. Below is the full code that provisions all necessary resources, including the web app, storage, Cosmos DB, Azure Search, and Cognitive Services.

### Complete Terraform Code Example

    # Create a resource group for the AI use case
    resource "azurerm_resource_group" "ai_usecase_rg" {
        name     = "${var.name}-rg"
        location = var.resource_group_location
    }
    
    # Create a service plan for hosting the web app and function apps
    resource "azurerm_service_plan" "service_plan" {
        name                = "${var.name}-plan"
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        location            = azurerm_resource_group.ai_usecase_rg.location
        os_type             = "Linux"
        sku_name            = "B1"
    }
    
    # Create a web app for hosting the AI web app
    resource "azurerm_linux_web_app" "web_app" {
        name                      = "${var.name}-web-app"
        location                  = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name       = azurerm_resource_group.ai_usecase_rg.name
        service_plan_id           = azurerm_service_plan.service_plan.id
    
        identity {
            type = "SystemAssigned"
        }
    
        site_config {
            always_on = false
        }
    
        depends_on = [
            azurerm_service_plan.service_plan
        ]
    }
    
    # Create a storage account for the web app
    resource "azurerm_storage_account" "storage" {
        name                     = var.storage_account_name
        resource_group_name      = azurerm_resource_group.ai_usecase_rg.name
        location                 = azurerm_resource_group.ai_usecase_rg.location
        account_tier             = "Standard"
        account_replication_type = "LRS"
        account_kind             = "StorageV2"
    }
    
    # Create a storage queue for the web app
    resource "azurerm_storage_queue" "queue" {
        name                  = "${var.name}-queue"
        storage_account_name  = azurerm_storage_account.storage.name
    }
    
    # Assign the Storage Queue Data Contributor role to the web app managed identity
    resource "azurerm_role_assignment" "app_service_queue_contributor" {
        principal_id         = azurerm_linux_web_app.web_app.identity[0].principal_id
        role_definition_name = "Storage Queue Data Contributor"
        scope                = azurerm_storage_account.storage.id
        
        depends_on = [
            azurerm_linux_web_app.web_app,
            azurerm_storage_queue.queue
        ]
    }
    
    # Create a storage container for the web app
    resource "azurerm_storage_container" "blob_container" {
        name                  = "${var.name}-blob-container"
        storage_account_name  = azurerm_storage_account.storage.name
        container_access_type = "private"
    }
    
    # Assign the Storage Blob Data Contributor role to the web app managed identity
    resource "azurerm_role_assignment" "app_service_blob_contributor" {
        principal_id         = azurerm_linux_web_app.web_app.identity[0].principal_id
        role_definition_name = "Storage Blob Data Contributor"
        scope                = azurerm_storage_account.storage.id
    
        depends_on = [
            azurerm_linux_web_app.web_app,
            azurerm_storage_container.blob_container
        ]
    }
    
    # Create a storage account for the function apps file system
    resource "azurerm_storage_account" "functions_file_system_storage" {
        name                     = var.functions_file_system_storage_account_name
        resource_group_name      = azurerm_resource_group.ai_usecase_rg.name
        location                 = azurerm_resource_group.ai_usecase_rg.location
    
    
        account_tier             = "Standard"
        account_replication_type = "LRS"
        account_kind             = "StorageV2"
    }
    
    # Create function apps for orchestration
    resource "azurerm_linux_function_app" "functions_app" {
        for_each = var.function_app
    
        name                       = each.value.name
        resource_group_name        = azurerm_resource_group.ai_usecase_rg.name
        location                   = azurerm_resource_group.ai_usecase_rg.location
        service_plan_id            = azurerm_service_plan.service_plan.id
        storage_account_name       = azurerm_storage_account.functions_file_system_storage.name
        storage_account_access_key = azurerm_storage_account.functions_file_system_storage.primary_access_key
    
        site_config {}
    
        depends_on = [
            azurerm_service_plan.service_plan,
            azurerm_linux_web_app.web_app
        ]
    }
    
    # Create a cognitive account with document intelligence model
    resource "azurerm_cognitive_account" "document-intelligence" {
        name                = "${var.name}-document-intelligence"
        location            = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        kind                = "FormRecognizer"
    
        sku_name = "S0"
    }
    
    # Create a user-assigned identity for the Cosmos DB account
    resource "azurerm_user_assigned_identity" "identity" {
        name                = "${var.name}-cosmosDBidentity"
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        location            = azurerm_resource_group.ai_usecase_rg.location
    }
    
    # Create a Cosmos DB account for storing metadata
    resource "azurerm_cosmosdb_account" "cosmosDB" {
        name                  = "${var.name}cosmosdb"
        location              = azurerm_resource_group.ai_usecase_rg.location
        resource_group_name   = azurerm_resource_group.ai_usecase_rg.name
        default_identity_type = join("=", ["UserAssignedIdentity", azurerm_user_assigned_identity.identity.id])
        offer_type            = "Standard"
        kind                  = "MongoDB"
    
        capabilities {
            name = "EnableMongo"
        }
    
        consistency_policy {
            consistency_level = "Strong"
        }
    
        geo_location {
            location          = "westeurope"
            failover_priority = 0
        }
    
        identity {
            type         = "UserAssigned"
            identity_ids = [azurerm_user_assigned_identity.identity.id]
        }
    }
    
    # Create a search service for indexing
    resource "azurerm_search_service" "search" {
        name                = "${var.name}-search"
        resource_group_name = azurerm_resource_group.ai_usecase_rg.name
        location            = azurerm_resource_group.ai_usecase_rg.location
        sku                 = "free"
        replica_count       = 1
        partition_count     = 1
    }

## Steps to Deploy

* **Define Variables**: Customize your `variables.tf` file with appropriate values such as the resource names, locations, and storage account details.
* **Initialize Terraform**: Run `terraform init` to initialize the working directory.
* **Preview Changes**: Use `terraform plan` to see what resources will be created and review the proposed changes.
* **Apply Configuration**: Execute `terraform apply` to deploy the resources on Azure.
* **Verify Deployment**: Go to the Azure Portal and check that all resources (Web App, Storage, Functions, Cosmos DB, etc.) have been correctly provisioned.
* **Configure Application**: Deploy your web application and functions code to the respective services (Azure Web App, Azure Functions).

## Conclusion

Deploying an AI-powered application on Azure using Terraform streamlines the provisioning process and ensures consistency across environments. By leveraging services like Azure App Service, Azure Functions, Cognitive Services, Cosmos DB, and Azure Search, you can build scalable and intelligent applications that meet modern demands.

### Key Takeaways

* **Infrastructure as Code**: Terraform allows you to define and manage your infrastructure efficiently.
* **Azure Integration**: Azure services provide powerful capabilities for AI applications.
* **Security Best Practices**: Using managed identities and role assignments enhances security.
* **Modularity**: Organizing resources logically makes management easier.

### Complete solutions

Code can be find in this GitHub repository: [dxdx-lhalicki/use\_azure\_ai (github.com)](https://github.com/dxdx-lhalicki/use_azure_ai)
