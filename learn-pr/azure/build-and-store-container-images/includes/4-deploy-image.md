Container images can be pulled from Azure Container Registry using many container management platforms, such as Azure Container Instances, Azure Kubernetes Service, and Docker for Windows or Mac. When running container images from Azure Container Registry, authentication credentials may be needed.

It is recommended to use an Azure service principal for authentication with Container Registry. It is also recommended to secure the Azure service principal credentials in Azure Key Vault. However, for our live practice we will use the built-in admin account that can be enabled in all Azure Container Registries. The admin account works with the free sandbox resources.

<!-- Activate the sandbox -->
[!include[](../../../includes/azure-sandbox-activate.md)]

Create a variable with the name of your container registry in lowercase (for example, instead of "MyContainer" make the value "mycontainer"). This variable is used throughout this unit.

```azurecli
ACR_NAME=<acrName>
```

### Admin Account

Azure Container Registries come with a built-in Admin account. This isn't tied to Azure AD or role based access controls and therefore **should only be used for testing**.

Enable the Admin Account.

```azurecli
az acr update -n $ACR_NAME --admin-enabled true
```

Query to get the autogenerated username and password

```azurecli
   az acr credential show --name $ACR_NAME
```

The output is similar to below. Take note of the `username` and the `value` paired with the `name` "password". You'll save these into key vault.

```output
{  "passwords": [
    {
      "name": "password",
      "value": "aaaaa"
    },
    {
      "name": "password2",
      "value": "bbbbb"
    }
  ],
  "username": "ccccc"
}
```

### Save the username and password to the key vault

Create an Azure Key Vault with the `az keyvault create` command.

```azurecli
az keyvault create --resource-group <rgn>[sandbox resource group name]</rgn> --name $ACR_NAME-keyvault
```

Use the `az keyvault secret set` command to store the username for the ACR in the vault. If you were using service principals you'd use the appId for this value. Since we're using the admin account this time we'll save the username from our query above. Enter the command below, don't forget to replace `<username>`.

```azurecli
az keyvault secret set --vault-name $ACR_NAME-keyvault --name $ACR_NAME-pull-usr --value <username>
```

Use the `az keyvault secret set` command to store the *password* in the vault. Replace `<password>` with the `password` from the query above.

```azurecli
az keyvault secret set --vault-name $ACR_NAME-keyvault --name $ACR_NAME-pull-pwd --value <password>
```

You've now created an Azure key vault and stored two secrets in it:

* `$ACR_NAME-pull-usr`: the container registry **username**.
* `$ACR_NAME-pull-pwd`: the container registry **password**.

You can now reference these secrets by name when you or your applications and services pull images from the registry.

### Deploy a container with Azure CLI

Now that the service principal credentials are stored in Azure Key Vault, your applications and services can use them to access your private registry.

Execute the following `az container create` command to deploy a container instance. The command uses the username and password stored in Azure Key Vault to authenticate to your container registry.

```azurecli
az container create \
    --resource-group <rgn>[sandbox resource group name]</rgn> \
    --name acr-tasks \
    --image $ACR_NAME.azurecr.io/helloacrtasks:v1 \
    --registry-login-server $ACR_NAME.azurecr.io \
    --ip-address Public \
    --location eastus \
    --registry-username $(az keyvault secret show --vault-name $ACR_NAME-keyvault --name $ACR_NAME-pull-usr --query value -o tsv) \
    --registry-password $(az keyvault secret show --vault-name $ACR_NAME-keyvault --name $ACR_NAME-pull-pwd --query value -o tsv)
```

Get the IP address of the Azure container instance.

```azurecli
az container show --resource-group  <rgn>[sandbox resource group name]</rgn> --name acr-tasks --query ipAddress.ip --output table
```

Open a browser and navigate to the IP address of the container. If everything has been configured correctly, you should see the following results:

![Sample web application with the text Hello World](../media/hello.png)
