# drupal-on-azure-container-apps

### Set up the environment

1. Sign in to the Azure CLI.

```bash
az login
```

1. Set up environment variables used in various commands to follow.

```bash
export RESOURCE_GROUP="my-drupal-apps-group"
export ENVIRONMENT_NAME="my-drupal-storage-environment"
export LOCATION="eastus"
```

1. Ensure you have the latest version of the Container Apps Azure CLI extension.

```bash
az extension add -n containerapp --upgrade
```

1. Register the Microsoft.App namespace.

```bash
az provider register -n Microsoft.App
```

1. Register the Microsoft.OperationalInsights provider for the Azure Monitor Log Analytics workspace if you haven't used it before.

```bash
az provider register -n Microsoft.OperationalInsights
```

### Create a Container Apps environment

1. Create a resource group.

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query "properties.provisioningState"
```

2. Create a Container Apps environment.

```bash
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location "$LOCATION" \
  --query "properties.provisioningState"
```

### Set up a storage account

1. Define a storage account name.

```bash
RANDOM==$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c10 ; echo '')
STORAGE_ACCOUNT_NAME="mydrupalstorageaccount$RANDOM"
```

1. Create an Azure Storage account.

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

1. Define a file share name.

```bash
FILE_SHARE_NAME="mydrupalfileshare"
```

1. Create a file share.

```bash
az storage share create --name $FILE_SHARE_NAME \
    --account-name $STORAGE_ACCOUNT_NAME --only-show-errors --output table
```

1. Get the storage account key.

```bash
STORAGE_ACCOUNT_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT_NAME \
  --query "[0].value" \
  --output tsv)
```

1. Define the storage mount name.

```bash
STORAGE_MOUNT_NAME="mydrupalstoragemount"
```

1. Create a storage mount.

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

### Create “Azure Database for MariaDB Servers” and a database instance

1. Drupal DB Settings

```bash
export DB_SERVER_NAME=drupal-db-srv-$RANDOM  # Must be globally unique, ie: 'drupal-db-srv-<unique>'
export DRUPAL_DB_HOST=$DB_SERVER_NAME.mariadb.database.azure.com
export DB_SERVER_SKU=GP_Gen5_2 # Azure Database for MariaDB SKU
export DRUPAL_DB_USER=myAdmin  # Cannot be 'admin'.
export DRUPAL_DB_PASSWORD=Zx3$RANDOM # Must include uppercase, lowercase, and numeric
export DRUPAL_DB_NAME=drupal_db
```

1. Create a MariaDB Server

```bash
echo "Creating Azure Database for MariaDB server '$DB_SERVER_NAME'."
az mariadb server create --name $DB_SERVER_NAME \
    --location $LOCATION --resource-group $RESOURCE_GROUP \
    --sku-name $DB_SERVER_SKU --ssl-enforcement Disabled \
    --version 10.3 --admin-user $DRUPAL_DB_USER \
    --admin-password $DRUPAL_DB_PASSWORD --output none
```

1. Enable Azure services (ie: Web App) to connect to the server.

```bash
echo "Configuring firewall on '$DB_SERVER_NAME' to allow access from Azure Services"
az mariadb server firewall-rule create --name AllowAllWindowsAzureIps \
    --resource-group $RESOURCE_GROUP --server-name $DB_SERVER_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 --output none
```

1. Create a blank DB for Drupal (Drupal's initialization process expects it to already exist and be empty.)

```bash
echo "Creating database '$DRUPAL_DB_NAME' on server '$DB_SERVER_NAME'."
az mariadb db create --name $DRUPAL_DB_NAME --server-name $DB_SERVER_NAME \
    --resource-group $RESOURCE_GROUP --output none
```

### Create the container app

1. Define the container app name.

```bash
CONTAINER_APP_NAME="my-drupal-app"
```

1. Create a container app.

```bash
az containerapp create -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
    --environment $ENVIRONMENT_NAME \
    --yaml container.yaml
```