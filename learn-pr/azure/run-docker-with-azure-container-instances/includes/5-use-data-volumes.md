By default, Azure Container Instances is stateless. If the container crashes or stops, all of its state is lost. To persist state beyond the lifetime of the container, you must mount a volume from an external store.

Here, you'll mount an Azure file share to an Azure container instance to you can store data and access it later.

## Create an Azure file share

Here you'll create a storage account and a file share that you'll later make accessible to an Azure container instance.

1. Your storage account requires a unique name. For learning purposes, run the following command to store a unique name in a Bash variable.

    ```bash
    ACI_PERS_STORAGE_ACCOUNT_NAME=mystorageaccount$RANDOM
    ```

1. Run the following `az storage account create` command to create your storage account.

    ```azurecli
    az storage account create \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --name $ACI_PERS_STORAGE_ACCOUNT_NAME \
      --sku Standard_LRS \
      --location eastus
    ```

1. Run the following command to place the storage account connection string into an environment variable named `AZURE_STORAGE_CONNECTION_STRING`.

    ```azurecli
    export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --name $ACI_PERS_STORAGE_ACCOUNT_NAME \
      --output tsv)
    ```

    `AZURE_STORAGE_CONNECTION_STRING` is a special environment variable that's understood by the Azure CLI. The `export` part makes this variable accessible to other CLI commands you'll run shortly.

1. Run this command to create a file share, named **aci-share-demo**, in the storage account.

    ```azurecli
    az storage share create --name aci-share-demo
    ```

## Get storage credentials

To mount an Azure file share as a volume in Azure Container Instances, you need these three values:

* The storage account name
* The share name
* The storage account access key

You already have the share name, **aci-share-demo**. Here you'll get the other two values.

1. Run the following command to get the storage account name.

    ```azurecli
    STORAGE_ACCOUNT=$(az storage account list \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --query "[?contains(name,'$ACI_PERS_STORAGE_ACCOUNT_NAME')].[name]" \
      --output tsv)
    ```

1. As an optional step, print the storage account name to the console.

    ```bash
    echo $STORAGE_ACCOUNT
    ```

1. Run the following command to get the storage account key.

    ```azurecli
    STORAGE_KEY=$(az storage account keys list \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --account-name $STORAGE_ACCOUNT \
      --query "[0].value" \
      --output tsv)
    ```

1. As an optional step, print the storage key to the console.

    ```bash
    echo $STORAGE_KEY
    ```

## Deploy a container and mount the file share

To mount an Azure file share as a volume in a container, you specify the share and volume mount point when you create the container.

1. Run this `az container create` command to create a container that's mounts `/aci/logs/` to your file share.

    ```azurecli
    az container create \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --name aci-demo-files \
      --image microsoft/aci-hellofiles \
      --location eastus \
      --ports 80 \
      --ip-address Public \
      --azure-file-volume-account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
      --azure-file-volume-account-key $STORAGE_KEY \
      --azure-file-volume-share-name aci-share-demo \
      --azure-file-volume-mount-path /aci/logs/
    ```

1. Run `az container show` to get your container's public IP address.

    ```azurecli
    az container show \
      --resource-group <rgn>[sandbox resource group name]</rgn> \
      --name aci-demo-files \
      --query ipAddress.ip \
      --output tsv
    ```

1. From a browser, navigate to your container's IP address. You see this.

    ![Azure Container Instances file share demo](../media/5-files-ui.png)

1. Enter some text into the form and click **Submit**. This action creates a file that contains the text you entered in the Azure file share.

1. Run this `az storage file list` command to display the files that are contained in your file share.

    ```azurecli
    az storage file list -s aci-share-demo -o table
    ```

1. Run `az storage file download` to download a file to your Cloud Shell session. Replace **\<filename\>** with one of the files that appeared in the previous step.

    ```azurecli
    az storage file download -s aci-share-demo -p <filename>
    ```

1. Run the `cat` command to print the contents of the file.

    ```azurecli
    cat <filename>
    ```

Remember that your data persists when your container exits. You can mount your file share to other container instances to make that data available to them.