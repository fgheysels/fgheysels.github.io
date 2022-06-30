---
layout: post
title: Getting started with Azure Connected Registry
comments: true
---

In 2021, Microsoft released a feature of Azure Container Registry called 'connected registry' in public preview.
A connected registry allows you to install a container registry on-prem which synchronizes or mirrors with an Azure Container Registry in the cloud.  This allows you to have your container images nearby which is beneficial in scenario's where you have an occasional or limited connection with the cloud.  At the moment that you want to pull a container image, the image doesn't need to be pulled from Azure, but is already present closer to where you need it.
[Here](https://docs.microsoft.com/en-us/azure/container-registry/intro-connected-registry), you can find more information regarding the connected registry container workloads.

I have explained in [another blogpost](https://www.codit.eu/blog/how-acrs-connected-registry-feature-helps-us-shipping-containers/) how we use connected registry to bring containers on board of vessels.

In this article, I'll go a bit deeper in the technicalities on how to deploy a connected registry.

# Installing a connected registry

## Define a connected registry instance in Azure

When you want to make use of an on-prem connected registry, you first need to define a 'connected registry' resource in Azure.
This can be done by using the `az acr connected-registry` [command](https://docs.microsoft.com/en-us/cli/azure/acr/connected-registry?view=azure-cli-latest).

When creating a connected registry, you must specify which repositories from the parent Azure Container Registry must be synced or mirrored.  This is explained in great detail [here](https://docs.microsoft.com/en-us/azure/container-registry/overview-connected-registry-access).

### Specify the repositories that must be synced

If you don't have a fixed list of repositories that must be synced with your connected registries, but rather want to sync all repositories that exist in a certain namespace, it can be quite cumbersome to manually list them all.

Instead, you can list all the repositories that you want to sync via scripting, for instance:

```powershell
$azureContainerRegistry = "mycontainerregistry"

# Get a list of all repositories that exist in the container registry.
# We'll use this list to retrieve the repositories that we want to sync
$existingRepositories = az acr repository list --name $azureContainerRegistry | ConvertFrom-Json

# In this example, we'll specify that we will sync all repositories that are defined in the 'somenamespace' namespace
$repositoriesToSync = $existingRepositories | Where-Object {$_ -Like "somenamespace/*"}

# When specifying the repositories that must be synced, each repository must be enclosed in double quotes and separated with a space
$repositoriesToSyncString = $repositoriesToSync | % {"""$_"""} | Join-String -Separator " "
```

### Create the connected registry Azure Resource

Once we have all the required information, the connected registry resource in Azure can be created.

```powershell
# Create the connected registry instance.  Use the invoke-expression command, because the $repositoriesToSyncString variable otherwise gives some issues
$connectedRegistryName = "myconnectedregistry"

Invoke-Expression "az acr connected-registry create --name $connectedRegistryName --registry $azureContainerRegistry --mode ReadOnly --sync-schedule '30 * * * *' --sync-window PT1H --repository $repositoriesToSyncString"
```

With the above command, we specify that the connected registry must sync all repositories that are listed in the `$repositoriesToSyncString`.
The connected registry can only read from the parent ACR since the mode is set to `ReadOnly`.
Syncing happens every hour at minute 30 and the sync window is set to 1 hour.

## Deploy the connected registry on-prem components

Now that we have created the necessary Azure resources for the connected registry, it is time to deploy the on-prem components.

In this article, we'll focus on deploying a connected registry in a Kubernetes cluster.  If you want to deploy a connected registry in IoT Edge, please see [this](https://docs.microsoft.com/en-us/azure/container-registry/quickstart-deploy-connected-registry-iot-edge-cli) article.

### Get the connection-string of the connected registry

Before we can deploy the connected registry components, we need to get hold of the connection-string of the connected registry resource that exists in Azure.
We will need this information when deploying the on-prem components, so let's just quickly retrieve the required information.

```powershell
$credentials = az acr connected-registry get-settings `
                      --name $connectedRegistryName `
                      --registry $azureContainerRegistry `
                      --parent-protocol https
                      --generate 1 `
                      --yes | ConvertFrom-Json
```

With the above command we generate a password and retrieve the credentials with one single command.
The `$credentials` object contains a connection-string that we'll use in the next step.

### Deploying connected registry service

When you want to pull container images from a connected registry in a Kubernetes cluster, you will need to deploy the connected registry components inside your Kubernetes container.
This can easily be done by installing a Helm chart provided by Microsoft.

I like to deploy these components in a dedicated Kubernetes namespace.  Execute the command below to create a Kubernetes namespace in an idempotent way.  This means that, when the namespace already exists, no error will be given:

```powershell
kubectl create namespace connected-registry --dry-run=client -o yaml | kubectl apply -f -
```

Getting the Helm chart that is provided by Microsoft is done by executing these commands:

```bash
helm chart pull mcr.microsoft.com/acr/connected-registry/chart:0.2.0
helm chart export mcr.microsoft.com/acr/connected-registry/chart:0.2.0 --destination $home
```

Once the Helm chart is pulled and exported, we can deploy the components:

```bash
helm upgrade -n connected-registry --install --wait `
                --set connectionString="$($credentials.ACR_REGISTRY_CONNECTION_STRING)" `
                --set httpEnabled=true `
                --set sync.chunkSizeInMb="1" `
                --set pvc.storageClassName="local-path" --set pvc.storageRequest="20Gi" connected-acr ./connected-registry 
```

Verify if everything is running correctly by executing `kubectl get pods -n connected-registry`.

As you can see, with the above command we specify the connection-string to the connected-registry resource that is defined in Azure.
We also specify that we want to be able to pull images from a private registry using http instead of being required to pull images using https (`httpEnabled=true`).

With the `sync.chunkSizeInMb` parameter you can specify the size of the chunks in which container layers must be downloaded.  If you're running on a slow or unstable connection, it is advised to set this parameter quite low.

The `pvc.storageClassName` specifies that we want to store the container images on local storage of the node.  The values that you can specify for this parameter vary per Kubernetes flavor.  The `local-path` value is a setting that is specific for [Rancher's K3s](https://rancher.com/docs/k3s/latest/en/).

### Configure pulling from a private registry

When you want to pull from a private registry using http instead of https, you need to configure that the connected-registry is a private registry.  
How this is done again depends on your Kubernetes version.  For K3s, this is done by editting the `/etc/rancher/k3s/registries.yaml` file.  More information on this can be found [here](https://rancher.com/docs/k3s/latest/en/installation/private-registry/).

To configure this, you'll need the IP address at which the connected registry is reachable.
The command below can be used to retrieve the IP address of the connected registry service that is running in Kubernetes:

```bash
$connectedRegistryIp = kubectl get service $connectedRegistryName -o=jsonpath='{@.spec.clusterIP}' -n connected-registry
```

Modify the `/etc/rancher/k3s/registries.yaml` file so that it looks like the sample below.  Of course, you need to use the IP address that you've acquired (`$connectedRegistryIp`) instead of the sample address (`10.1.1.1`)

```yaml
mirrors:
  "10.1.1.1:80":
    endpoint: 
    - "http://10.1.1.1:80"

configs:
  "10.1.1.1:80":
    tls:
    - insecure_skip_verify: true
```

Modifying this file can also be automated using your favorite scripting language.  Using Powershell, you can for instance use the [`powershell-yaml`](https://github.com/cloudbase/powershell-yaml) module to do that.
Here is the code to do that:

```powershell
Install-Module -Name powershell-yaml -Force
Import-Module powershell-yaml -Force

$mirror = @{ endpoint = @("http://$($connectedRegistryIp):80")}
$config = @{ tls = @{ insecure_skip_verify = $true }}

$contents = @{}

if ($True -eq (Test-Path /etc/rancher/k3s/registries.yaml) )
{
    $contents = Get-Content -Path /etc/rancher/k3s/registries.yaml | ConvertFrom-Yaml
}
else 
{
    # If the file does not exist, create it.  This is necessary to avoid errors
    # later in the script where we'll (temporarly) modify the access rights on that file.
    sudo touch /etc/rancher/k3s/registries.yaml
}

## Add entries for the connected ACR
if( $null -eq $contents["mirrors"] )
{
    $contents["mirrors"] = @{}
}
$contents["mirrors"]["""$($connectedRegistryIp):80"""] = $mirror

if( $null -eq $contents["configs"] )
{
    $contents["configs"] = @{}
}
$contents["configs"]["""$($connectedRegistryIp):80"""] = $config

sudo -E chmod a+w /etc/rancher/k3s/registries.yaml || true

$yamlContent = $contents | ConvertTo-Yaml
## It seems that there's a bug in ConvertTo-Yaml: quotes are escaped with an additional single quote
$yamlContent = $yamlContent -Replace "'""", '"' -Replace """'", '"'
Set-Content -Path /etc/rancher/k3s/registries.yaml -Value $yamlContent

sudo -E chmod 644 /etc/rancher/k3s/registries.yaml || true
```

After the `registries.yaml` file has been modified, K3s must be restarted for the changes to have effect:

```bash
sudo -E systemctl restart k3s
```

The connected registry is now in place and will start pulling container images from it's parent Azure Container Registry.

# Pulling images from a connected registry

Now that the connected registry components are up and running, we need to configure it so that images can be pulled from the connected registry.

## Configure Client Token

Before images can be pulled from the connected registry, a client token must be configured.  As explained [here](https://docs.microsoft.com/en-us/azure/container-registry/overview-connected-registry-access#client-tokens), the client token specifies from which repositories images can be pulled.

A token is created using a [scope map](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-repository-scoped-permissions).  In the scope map, we specify the repositories from which images can be pulled.

We can use the `$existingRepositories` that we have declared and initialized earlier for this.  We can extract the repositories that the client is allowed to pull.
Again, let us specify that images can be pulled if they belong to a repository that is in a certain namespace:

```powershell
$allowedRepositories = $existingRepositories | Where-Object {$_ -Like "somenamespace/*"} 
```

As can be read from the [documentation](https://docs.microsoft.com/en-us/cli/azure/acr/scope-map?view=azure-cli-latest#az-acr-scope-map-create) of the `az acr scope-map create` command, each repository must be prefixed with `--repository` and must be suffixed with the actions that are allowed for that repository.
The following command does that:

```powershell
$allowedRepositoriesString = $allowedRepositories | % { "--repository $_ content/read" } | Join-String -Separator " "
```

Now we have everything to create scope map.  Again, we use the `Invoke-Expression` command to avoid problems with the quotes in the `$allowedRepositoriesString` variable:

```powershell
$scopemapName = "myscopemap"

Invoke-Expression "az acr scope-map create -n $scopemapName -r $azureContainerRegistryName --description 'Scope map for connected registry client token' $allowedRepositoriesString"
```

Now, we can use the scope map to generate a client access token and add that token to our connected registry:

```powershell
$tokenName  = "myclienttoken"

$token = az acr token create -n $tokenName -r $azureContainerRegistryName --scope-map $scopemapName | ConvertFrom-Json

az acr connected-registry update --registry $azureContainerRegistryName --name $connectedRegistryName --add-client-tokens $tokenName
```

> Note that it is a good idea to often roll the client access token. This can be done by simply executing the above commands again.

## Create a Pull Secret for the Connected Registry

When you want to pull images in Kubernetes, you need a pull secret.  The `$token` that we have just generated, can be used to create a pull secret for the connected registry:

```powershell
$pullSecretUserName = $token.credentials.username
$pullSecretPassword = $token.credentials.passwords[0].value

# Retrieve IP address of connected-registry service that is running in k3s
$connectedRegistryIpAddress = kubectl get service $connectedRegistryName -o=jsonpath='{@.spec.clusterIP}' -n connected-registry

$connectedRegistryUrl = "http://$($connectedRegistryIpAddress):80"
                    
kubectl delete secret mypullsecretname --ignore-not-found -n mynamespace
kubectl create secret docker-registry mypullsecretname --docker-server=$connectedRegistryUrl --docker-username=$pullSecretUserName --docker-password=$pullSecretPassword -n mynamespace
```

Now you can use this pull secret in your Kubernetes deployments to pull container images from your Connected Registry!
Just don't forget to specify the pull secret in your Kubernetes deployment.yaml manifest.

I hope this article helps you in setting up an Azure Connected Registry!

Frederik