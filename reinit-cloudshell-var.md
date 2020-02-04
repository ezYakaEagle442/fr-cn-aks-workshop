```sh
subName="Your NAME Internal subscription"

#subName=$(az account list --query "[?name=='${subName}'].{name:name}"  --output tsv)
echo "subscription Name :" $subName 
subId=$(az account list --query "[?name=='${subName}'].{id:id}"  --output tsv)
echo "subscription ID :" $subId

az account set --subscription $subId
az account show 

location=francecentral 
target_namespace="staging"
appName="frcnstu" 
cluster_name="aks-${appName}-${target_namespace}-101" #aks-<App Name>-<Environment>-<###>
version=$(az aks get-versions -l $location --query 'orchestrators[-4].orchestratorVersion' -o tsv) 
echo "version is :" $version 

custom_dns="msfrancerocks.fr"
dnz_zone="cloudapp.net" # azurewebsites.net 
storage_name="stne""${appName,,}"
git_url="https://github.com/spring-projects/spring-petclinic.git"
network_plugin="azure"
network_policy="azure"
rg_name="rg-${appName}-${location}" 
appgwName="ingress-appgw"
node_pool_name="devnodepool"
vnet_name="vnet-${appName}"
subnet_name="snet-${appName}"
vault_name="vault-${appName}"
vault_secret="NoSugarNoStar" 
analytics_workspace_name="${appName}AnalyticsWorkspace"
analytics_workspace_template="deployworkspacetemplate.json"
acr_registry_name="acr${appName,,}"

source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc 
alias k=kubectl
complete -F __start_kubectl k

sp_password=`cat spp.txt`
sp_id=`cat spid.txt`
#sp_id=$(az ad sp list --all --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
#sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
echo "Service Principal ID:" $sp_id  
az ad sp show --id $sp_id

vnet_id=$(az network vnet show --resource-group $rg_name --name  $vnet_name --query id -o tsv)
subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name  $vnet_name  --name $subnet_name --query id -o tsv)
echo "VNet Id :" $vnet_id	
echo "Subnet Id :" $subnet_id	

az aks get-credentials --resource-group $rg_name  --name $cluster_name
kubectl cluster-info

acr_registry_id=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "id" --output tsv)
echo "ACR registry ID :" $acr_registry_id
az acr repository list  --name $acr_registry_name # --resource-group $rg_name
az acr check-health --yes -n $acr_registry_name 

kubectl get secrets -n $target_namespace
k get svc -n $target_namespace

ing_ctl_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
kubectl get endpoints petclinic-internal-service -n $target_namespace


```

