# DevOps Tools - Terraform, Serveless, Azure e Telemetria

## Requisitos

- Terraform ">= 0.12"
- Conta no GitHub
- NodeJS > 10
- Azure Cli > 2\*

## Estrutura do projeto

## Infraestrutura

    https://github.com/aferlim/ltm-serverless-poc-iac

### Passo 1

    mkdir serverless-hands-on && cd serverless-hands-on
    mkdir iac && cd iac

### Passo 2

Abrir o Visual Studio Code

    code .

### Passo 3 - Estado Inicial

Dentro do projeto no VsCode, criar uma pasta chamada **start-state.**

Criar o arquivo de versões (Apenas convenção) e adicionar o conteúdo (Dentro da pasta start-state)

version.tf

    terraform {
        required_version = ">= 0.12"
    }

Utilizaremos o provider do Azure e agora precisamos
configurar o provider, para isso crie o arquivo _provider.tf_

    provider "azurerm" {
        version = "~> 2.4.0"
        features {}

    }

Criar arquivo de variáveis _variables.tf_

    variable "AZ_REGION" {
        default = "East US"
    }

    variable "Resource_Group" {
        default = "terraform-state"
    }

Vamos criar o grupo do recursos do state dentro do Azure, para tal crie um arquivo **resource-group.tf**

    resource "azurerm_resource_group" "serverless-group" {
        name     = var.Resource_Group
        location = var.AZ_REGION

        tags = {
            environment = "staging"
        }
    }

Vamos criar o storage account e o storage container:

    resource "azurerm_storage_account" "tfstate_sa" {
        name                     = "tfstateltmserverlesspoc"
        resource_group_name      = azurerm_resource_group.serverless-group.name
        location                 = azurerm_resource_group.serverless-group.location
        account_tier             = "Standard"
        account_replication_type = "LRS"
        account_kind             = "BlobStorage"

        tags = {
            environment = "development"
        }
    }

    resource "azurerm_storage_container" "tfstate_container" {
        name                  = "tfstatecontainer"
        storage_account_name  = azurerm_storage_account.tfstate_sa.name
        container_access_type = "private"
    }

Criar o arquivo de variáveis de saída, para tal vamos criar o arquivo _output.tf_ e incluir o trecho:

    output "storage_account_name" {
        value = azurerm_storage_account.tfstate_sa.name
    }

    output "container_name" {
        value = azurerm_storage_container.tfstate_container.name
    }

    output "access_key" {
        value = azurerm_storage_account.tfstate_sa.primary_access_key
    }

Explorar alguns comandos terraform

    terraform output
    terraform workspace list
    terraform state list
    terraform show

### Passo 4 - Recursos a partir do estado remoto (Vamos utilizar a pasta raiz (IAC) para não haver integração entre a infra do estado e a infra da nossa solução)

Na raiz da aplicação, fora da pasta **start-state**, vamos iniciar nosso provider (provider.tf):

    provider "azurerm" {
        version = "~> 2.4.0"
        features {}
    }

Na raiz da aplicação vamos criar o arquivo **backend.tf** e iniciar a configuração do nosso estado remoto apontando para o storege account que acabamos de configurar no Azure:

    terraform {
        required_version = ">= 0.12"
        backend "azurerm" {
            resource_group_name  = "terraform-state"
            storage_account_name = "tfstateltmserverlesspoc"
            container_name       = "tfstatecontainer"
            key                  = "terraform.tfstate"
        }
    }

Configurar nosso arquivo de variáveis (variables.tf)

    variable "tf_state_resource_group" {
        default = "terraform-state"
    }

    variable "tf_storage_account_name" {
        default = "tfstateltmserverlesspoc"
    }

    variable "tf_container_name" {
        default = "tfstatecontainer"
    }

    variable "POC_Azure_Region" {
        default = "East US"
    }

    variable "POC_Resource_Group" {
        default = "terraform-serverless-poc"
    }

    variable "POC_StaticFile_Name" {
        default = "ltmserverlessstatic"
    }

    variable "AzSubscriptionId" {
        default = "660d3b8a-8752-4b02-87f5-e4b07c5ac69e"
    }

Resource Group (resource-group.tf)

    resource "azurerm_resource_group" "serverless-group" {
        name     = var.POC_Resource_Group
        location = var.POC_Azure_Region
    }

### Vamos utilizar o arquivo main.tf para incluir os recursos

Static webapp:

    resource "azurerm_storage_account" "static_website" {
        account_replication_type  = "GRS"
        account_tier              = "Standard"
        account_kind              = "StorageV2"
        location                  = azurerm_resource_group.serverless-group.location
        name                      = var.POC_StaticFile_Name
        resource_group_name       = azurerm_resource_group.serverless-group.name
        enable_https_traffic_only = true

        static_website {
            index_document     = "index.html"
            error_404_document = "error.html"
        }
    }

SignalR Service

    resource "azurerm_signalr_service" "serverless_signalr" {
        name                = "serverlesspoc-signalr"
        location            = azurerm_resource_group.serverless-group.location
        resource_group_name = azurerm_resource_group.serverless-group.name

        sku {
            name     = "Standard_S1"
            capacity = 2
        }

        cors {
            allowed_origins = ["*"]
        }

        features {
            flag  = "ServiceMode"
            value = "Serverless"
        }

        features {
            flag  = "EnableConnectivityLogs"
            value = "True"
        }
    }

Storage da Function

    resource "azurerm_storage_account" "function_storage" {
        name                     = "votepocstorage"
        location                 = azurerm_resource_group.serverless-group.location
        resource_group_name      = azurerm_resource_group.serverless-group.name
        account_tier             = "Standard"
        account_replication_type = "LRS"
    }

O Service Plan

    resource "azurerm_app_service_plan" "serverless_plan" {
        name                = "serverlessplan"
        location            = azurerm_resource_group.serverless-group.location
        resource_group_name = azurerm_resource_group.serverless-group.name

        sku {
            tier = "Dynamic"
            size = "Y1"
        }
    }

Telemetria:

    resource "azurerm_application_insights" "application_insights" {
        name                = "serverless-appInsights"
        location            = azurerm_resource_group.serverless-group.location
        resource_group_name = azurerm_resource_group.serverless-group.name
        application_type    = "web"

    }

Azure Function

    resource "azurerm_function_app" "vote_function" {
        version                   = "~2"
        name                      = "votepoc"
        location                  = azurerm_resource_group.serverless-group.location
        resource_group_name       = azurerm_resource_group.serverless-group.name
        app_service_plan_id       = azurerm_app_service_plan.serverless_plan.id
        storage_connection_string = azurerm_storage_account.function_storage.primary_connection_string

        app_settings = {
            "AzureSignalRConnectionString" : azurerm_signalr_service.serverless_signalr.primary_connection_string
            "WEBSITE_ENABLE_SYNC_UPDATE_SITE" : "true"
            "WEBSITE_RUN_FROM_PACKAGE" : "1"
            "APPINSIGHTS_INSTRUMENTATIONKEY" : azurerm_application_insights.application_insights.instrumentation_key
        }

        site_config {
            cors {
            allowed_origins     = ["*"]
            support_credentials = false
            }
        }
    }

Outputs (output.tf)

    output "Webstatic_secondary_endpoint" {
        value = azurerm_storage_account.static_website.secondary_web_endpoint
    }

    output "static_website_storage_primary_connection_string" {
        value = azurerm_storage_account.static_website.primary_connection_string
    }

    output "function_storage_storage_primary_connection_string" {
        value = azurerm_storage_account.static_website.primary_connection_string
    }

    output "serverless_signalr_primary_connection_string" {
        value = azurerm_signalr_service.serverless_signalr.primary_connection_string
    }

    output "azurerm_function_app_vote_function_default_hostname" {
        value = azurerm_function_app.vote_function.default_hostname
    }

### Adicione o arquivo gitignore na solução

O arquivo se encontra nesse repositório.

[gitignore.txt](/gitignore.txt)

Crie o arquivo .gitignore e cole o conteudo.

Crie um novo repositório no seu GitHub.

Configurando o git local:

    git remote add origin https://github.com/aferlim/seurepo.git

Comitando:

    git add .
    git commit -m "my first commit :rocket:"

Subindo as alterações:

    git push -u origin master

Para utilizar o Azure Cloud Shell, basta clonar o repositório no ambiente do Azure

### Verificando mudanças

Vá ao console da aplicação e execute, ou execute via azure shell:

    az login
    terraform plan

Aplicando:

    terraform apply

Destruindo:

    terraform destroy

Mais commandos terraform:

    terraform output
    terraform workspace list
    terraform state
    terraform -h

### Passo 5 - Configurando a Azure Function (Vá para a pasta raiz - serverless-hands-on)

1- Faça um fork do repositório e faça o push localmente:

https://github.com/aferlim/azure-function-signalr

2- Adicione o arquivo **local.settings.json** na pasta ServerlessPoc. O arquivo encontra-se na raiz desse repositório

3- Altere as configurações para as credenciais da sua conta do Azure.

Para saber as configurações, na pasta **iac** execute o commando:

    terraform output

### Configure o seu pipeline do Azure DevOps

4- Acesse o Azure DevOps com a sua conta criada:

https://dev.azure.com/

5 - Instale a ferramenta que utilizaremos para Code Coverage:

https://marketplace.visualstudio.com/items?itemName=Palmmedia.reportgenerator

6- Crie um novo projeto no azure DevOps

7- Adicione as Service Connections do Azure (Azure Resource Manager) e do GitHub.

8 - Configure o Continous Integration da Azure Function

9 - Configurar o Continuous Deployment

### Passo 6 - Front End

1- Faça um fork do repositório e abra localmente:

https://github.com/aferlim/azure-function-signalr

2 - Configurar as variaveis do arquivo

    Subscription needs a storage account and a website

    azureSubscription: "Visual Studio Professional (660d3b8a-8752-4b02-87f5-e4b07c5ac69e)"

    This needs to have a static website setup with the default container ($web)
    clientBlobAccountName: "ltmserverlessstatic"

    This is provided to the client app so it knows how to hit the right server
    functionEndpoint: "https://votepoc.azurewebsites.net"

4 - Commite as informações

Comitando:

    git add .
    git commit -m "my first commit :rocket:"

Subindo as alterações:

    git push -u origin master

3 - Configure o Continous Integration e o Continuous Deployment
