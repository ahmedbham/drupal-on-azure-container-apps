# Deploy Drupal on Azure Container Apps

Drupal is an open-source content management framework written in the PHP server-side scripting language. Drupal provides a backend framework for many enterprise websites, with standard features such as content authoring, reliable performance, and robust security.

The two code sets used by every Drupal site: the codebase and the database. The codebase is the set of files that make up Drupal itself, along with any themes, modules, or libraries you've added. The database is where Drupal stores most of its configuration and all of its content.

## Solution Architecture

![Solution Architecture](/solution-architecture.png)

- **Azure Container Apps (ACA)** for hosting Drupal application
- **Azure Database for MariaDB** for hosting Drupal database
- **Azure File Share** for hosting Drupal files

## Setup

Install the latest version of the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

- Sign in to the Azure CLI.

```bash
az login
```

* Set up environment variables used in various commands to follow (change according to your preference).

```bash
export RESOURCE_GROUP="my-drupal-apps-group"
export ENVIRONMENT_NAME="my-drupal-storage-environment"
export LOCATION="eastus"
```

* Ensure you have the latest version of the Container Apps Azure CLI extension.

```bash
az extension add -n containerapp --upgrade
```

* Register the Microsoft.App namespace.

```bash
az provider register -n Microsoft.App
```

* Register the Microsoft.OperationalInsights provider for the Azure Monitor Log Analytics workspace if you haven't used it before.

```bash
az provider register -n Microsoft.OperationalInsights
```

* Create a resource group.

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query "properties.provisioningState"
```

## Create ACA Environment

* Create ACA environment.

```bash
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location "$LOCATION" \
  --query "properties.provisioningState"
```

```bash
export ENVIRONMENT_ID=$(az containerapp env show \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "id" \
  --output tsv)

echo $ENVIRONMENT_ID
```

## Set up a storage account

* Define a storage account name (must be globally unique).

```bash
RAND=$(head /dev/urandom | tr -dc a-z0-9 | head -c10 ; echo '')
STORAGE_ACCOUNT_NAME="druplaca$RAND"

echo "Storage account name is" $STORAGE_ACCOUNT_NAME
```

* Create an Azure Storage account.

```bash
az storage account create \
  --resource-group $RESOURCE_GROUP \
  --name $STORAGE_ACCOUNT_NAME \
  --location "$LOCATION" \
  --kind StorageV2 \
  --sku Standard_LRS \
  --enable-large-file-share \
  --query provisioningState
```

* Define a file share name.

```bash
FILE_SHARE_NAME="mydrupalfileshare"
```

* Create a file share.

```bash
az storage share create --name $FILE_SHARE_NAME \
    --account-name $STORAGE_ACCOUNT_NAME --only-show-errors --output table
```

* Get the storage account key.

```bash
STORAGE_ACCOUNT_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT_NAME \
  --query "[0].value" \
  --output tsv)

echo "Storage Account Key is $STORAGE_ACCOUNT_KEY"
```

* Define the storage mount name.

```bash
export STORAGE_MOUNT_NAME="mydrupalstoragemount"
```

* Create a storage mount.

```bash
az containerapp env storage set \
  --access-mode ReadWrite \
  --azure-file-account-name $STORAGE_ACCOUNT_NAME \
  --azure-file-account-key $STORAGE_ACCOUNT_KEY \
  --azure-file-share-name $FILE_SHARE_NAME \
  --storage-name $STORAGE_MOUNT_NAME \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

## Create “Azure Database for MariaDB Servers” and a database instance

* Drupal DB Settings

```bash
export DB_SERVER_NAME=drupal-db-srv-$RAND  # Must be globally unique, ie: 'drupal-db-srv-<unique>'
export DRUPAL_DB_HOST=$DB_SERVER_NAME.mariadb.database.azure.com
export DB_SERVER_SKU=GP_Gen5_2 # Azure Database for MariaDB SKU
export DRUPAL_DB_USER=myAdmin  # Cannot be 'admin'.
export DRUPAL_DB_PASSWORD=Zx3$RAND # Must include uppercase, lowercase, and numeric
export DRUPAL_DB_NAME=drupal_db
```

* Create a MariaDB Server

```bash
az mariadb server create --name $DB_SERVER_NAME \
    --location $LOCATION --resource-group $RESOURCE_GROUP \
    --sku-name $DB_SERVER_SKU --ssl-enforcement Disabled \
    --version 10.3 --admin-user $DRUPAL_DB_USER \
    --admin-password $DRUPAL_DB_PASSWORD --output none
```

* Enable Azure services (ie: Web App) to connect to the server.

```bash
az mariadb server firewall-rule create --name AllowAllWindowsAzureIps \
    --resource-group $RESOURCE_GROUP --server-name $DB_SERVER_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 --output none
```

* Create a blank DB for Drupal (Drupal's initialization process expects it to already exist and be empty.)

```bash
az mariadb db create --name $DRUPAL_DB_NAME --server-name $DB_SERVER_NAME \
    --resource-group $RESOURCE_GROUP --output none
```

## Create the container app

* Define Azure Container App name.

```bash
export CONTAINER_APP_NAME="my-drupal-app"
```

* We will use a yaml template file `drupal-aca.yaml` to create the container app. This file has placeholders for following fields that will be set by executing `update-yaml.sh` script file.
    * `environmentId` - The environment ID from the previous step.
    * `storageMountName` - The storage mount name from the previous step.
    * `storageMountPath` - The path to the storage mount. This is the path that Drupal will use to store files.
    * `drupalDbHost` - The MariaDB server host name from the previous step.
    * `drupalDbUser` - The MariaDB server user name from the previous step.
    * `drupalDbPassword` - The MariaDB server password from the previous step.
    * `drupalDbName` - The MariaDB server database name from the previous step.
    * `minReplicas` - The minimum number of replicas for the container app.
    * `maxReplicas` - The maximum number of replicas for the container app.

* ACA allows you to create secrets that can be used in the yaml template file. We will create secrets for the following environment variables that are used in the yaml template file.
    * `DRUPAL_DB_HOST`
    * `DRUPAL_DB_USER`
    * `DRUPAL_DB_PASSWORD`
    * `DRUPAL_DB_NAME`

```bash
chmod +x update-yaml.sh
./update-yaml.sh
```

* Create a container app.

```bash
az containerapp create -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
    --environment $ENVIRONMENT_NAME \
    --yaml drupal-aca-updated.yaml \
    --query properties.configuration.ingress.fqdn
```

## Accessing Drupal site

The last command will return the FQDN of the container app. You can access the Drupal site by navigating to the FQDN in a browser. Click on the `Log in` link in the top right corner of the page. The default username is `user` and the password is `bitnami`.