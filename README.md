# fr-cn-aks-workshop
Cloud-Native Workshop

## Pre-requisites

You need an Azure subscription. If you do not have one, to get started quickly go to [https://my.visualstudio.com](https://my.visualstudio.com) and log in using your CorpNet login

For MS employees, ask help from the proctor to create your own internal subscription. 
 
Check your subscription at : [https://account.azure.com/subscriptions](https://account.azure.com/subscriptions) 
then go to [https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade](https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade ) --> remove filter "Show only subscriptions selected in the 
global subscriptions filter" to see it 

#### Azure Cloud Shell

You can use the Azure Cloud Shell accessible at <https://shell.azure.com> once you login with an Azure subscription.

#### Uploading and editing files in Azure Cloud Shell

- You can use `vim <file you want to edit>` in Azure Cloud Shell to open the built-in text editor.
- You can upload files to the Azure Cloud Shell by dragging and dropping them
- You can also do a `curl -o filename.ext https://file-url/filename.ext` to download a file from the internet.

### Naming conventions
See also [See also https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/naming-and-tagging](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/naming-and-tagging)

### Tools

You can use the Azure Cloud Shell accessible [https://portal.azure.com](https://portal.azure.com) once you login with an Azure subscription. 
The Azure Cloud Shell has the Azure CLI pre-installed and configured to connect to your Azure subscription as well as `kubectl` and `helm`.
```sh
az --version
az account list 
az account show 
# az extension remove --name aks-preview
# az extension add --name aks-preview
```

```sh
# https://kubernetes.io/docs/reference/kubectl/cheatsheet/
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc 
alias k=kubectl
complete -F __start_kubectl k
```

Optionnaly : If you want to run PowerShell
```sh
alias kn='kubectl config set-context --current --namespace '
#If you run kubectl in PowerShell ISE , you can also define aliases :
function k([Parameter(ValueFromRemainingArguments = $true)]$params) { & kubectl $params }
function kubectl([Parameter(ValueFromRemainingArguments = $true)]$params) { Write-Output "> kubectl $(@($params | ForEach-Object {$_}) -join ' ')"; & kubectl.exe $params; }
function k([Parameter(ValueFromRemainingArguments = $true)]$params) { Write-Output "> k $(@($params | ForEach-Object {$_}) -join ' ')"; & kubectl.exe $params; }
```

### Azure subscription

Please use your username and password to login to <https://portal.azure.com>.

Also please authenticate your Azure CLI by running the command below on your machine and following the instructions.

```sh
# /!\ In CloudShell, the default subscription is not always the one you thought ...
subName="set here the name of your subscription"

subName=$(az account list --query "[?name=='${subName}'].{name:name}"  --output tsv)
echo "subscription Name :" $subName 
subId=$(az account list --query "[?name=='${subName}'].{id:id}"  --output tsv)
echo "subscription ID :" $subId

az account set --subscription $subId
az account show 
az login
```

## Build AKS Cluster 

### Set-up environment variables

<span style="color:red">/!\ IMPORTANT </span> : your **appName** & **cluster_name** values MUST BE UNIQUE
```sh
# snippet sample at : https://github.com/Azure-Samples/openhack-devops-proctor/blob/master/provision-team/provision_aks.sh

# az account list-locations : francecentral | northeurope | westeurope
location=francecentral 
echo "location is : " $location 

target_namespace="staging"
echo "Target namespace:" $target_namespace

appName="frcnstu" 
echo "appName is : " $appName 

cluster_name="aks-${appName}-${target_namespace}-101" #aks-<App Name>-<Environment>-<###>
echo "Cluster name:" $cluster_name

# target : version 1.15.7
version=$(az aks get-versions -l $location --query 'orchestrators[-4].orchestratorVersion' -o tsv) 
echo "version is :" $version 


```

Note: The here under variables are built based on the varibales defined above, you should not need to modify them, just run this snippet

```sh
custom_dns="msfrancerocks.fr"
echo "Custom DNS Zone is : " $custom_dns

dns_zone="cloudapp.net" # azurewebsites.net 
echo "DNS Zone is : " $dns_zone

# Storage account name must be between 3 and 24 characters in length and use numbers and lower-case letters only
storage_name="stfr""${appName,,}"
echo "Storage name:" $storage_name

# original sources at https://github.com/spring-projects/spring-petclinic.git then forked to https://github.com/spring-petclinic
# https://stackoverflow.com/questions/31939849/spring-boot-default-log-location
# https://spring-petclinic.github.io/docs/forks.html

git_url="https://github.com/spring-projects/spring-petclinic.git"
echo "Project git repo URL : " $git_url 

network_plugin="azure"
echo "Network Plugin is : " $network_plugin 

network_policy="azure"
echo "Network Policy is : " $network_policy 

rg_name="rg-${appName}-${location}" 
echo "RG name:" $rg_name 

appgwName="ingress-appgw"
echo "App Gateway name:" $appgwName

# --nodepool-name can contain at most 12 characters. must conform to the following pattern: '^[a-z][a-z0-9]{0,11}$'.
node_pool_name="pocnodepool"
echo "Node Pool name:" $node_pool_name

vnet_name="vnet-${appName}"
echo "VNet Name :" $vnet_name

subnet_name="snet-${appName}"
echo "Subnet Name :" $subnet_name

vault_name="vault-${appName}"
echo "Vault name :" $vault_name

vault_secret="NoSugarNoStar" 
echo "Vault secret:" $vault_secret 

analytics_workspace_name="${appName}AnalyticsWorkspace"
echo "Analytics Workspace Name :" $analytics_workspace_name

analytics_workspace_template="deployworkspacetemplate.json"
echo "Analytics Workspace Name :" $analytics_workspace_template

acr_registry_name="acr${appName,,}"
echo "ACR registry Name :" $acr_registry_name

```

### Create Service Principal
```sh
# https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
# https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create-for-rbac
# https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest
# As of Azure CLI 2.0.68, the --password parameter to create a service principal with a user-defined password is no longer supported to prevent the accidental use of weak passwords.
sp_password=$(az ad sp create-for-rbac --name $appName --role contributor --query password --output tsv)
echo $sp_password > spp.txt
echo "Service Principal Password saved to ./spp.txt. IMPORTANT Keep your password ..." 
# sp_password=`cat spp.txt`
#sp_id=$(az ad sp list --all --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
echo "Service Principal ID:" $sp_id 
echo $sp_id > spid.txt
# sp_id=`cat spid.txt`
az ad sp show --id $sp_id
```

### Create RG & Networks
```sh
az group create --name $rg_name --location $location
# https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create
# https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction#types-of-storage-accounts
az storage account create --name $storage_name --kind StorageV2 --sku Standard_LRS --resource-group $rg_name --location $location --https-only true

az network vnet create --name $vnet_name --resource-group $rg_name --address-prefixes 172.16.0.0/16 --location $location
az network vnet subnet create --name $subnet_name --address-prefixes 172.16.1.0/24 --vnet-name $vnet_name --resource-group $rg_name 

vnet_id=$(az network vnet show --resource-group $rg_name --name  $vnet_name --query id -o tsv)
subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name  $vnet_name  --name $subnet_name --query id -o tsv)
echo "VNet Id :" $vnet_id	
echo "Subnet Id :" $subnet_id	
```

### Create AKS Cluster
For Advanced networking options, see https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni

```sh
az aks create --name $cluster_name \
    --resource-group $rg_name \
    --service-principal $sp_id \
    --client-secret $sp_password \
    --zones 1 2 3 \
    --vnet-subnet-id $subnet_id \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --location $location \
    --kubernetes-version $version \
    --node-count 3 \
    --generate-ssh-keys \
    --network-plugin $network_plugin \
    --network-policy $network_policy \
    --nodepool-name  $node_pool_name \
    --verbose \
    --load-balancer-sku standard \
    --vm-set-type VirtualMachineScaleSets
    # outboundType (public preview target on release) : Addresses SLB configuration to skip setup of public IP addresses and backend pools https://github.com/Azure/azure-rest-api-specs/blob/master/specification/containerservice/resource-manager/Microsoft.ContainerService/stable/2020-01-01/managedClusters.json#L1679
    #--outboundType=userDefinedRouting|loadBalancer

az aks get-credentials --resource-group $rg_name  --name $cluster_name
kubectl cluster-info
#kubectl config view

# Get the id of the service principal configured for AKS
CLIENT_ID=$(az aks show --resource-group $rg_name --name $cluster_name --query "servicePrincipalProfile.clientId" --output tsv)
echo "CLIENT_ID:" $CLIENT_ID 
```

### Create Namespaces
```sh
kubectl create namespace development
kubectl label namespace/development purpose=development

kubectl create namespace staging
kubectl label namespace/staging purpose=staging

kubectl create namespace production
kubectl label namespace/production purpose=production
```

### Optionnal Play: what resources are in your cluster

```sh
kubectl get nodes
kubectl describe namespace production

# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-node-distribution-across-zones
kubectl describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"

kubectl get pods
kubectl top node
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false

kubectl get roles --all-namespaces
kubectl get serviceaccounts --all-namespaces
kubectl get rolebindings --all-namespaces
kubectl get ingresses  --all-namespaces
```

### Optionnal: Monitor
```sh
az aks enable-addons --resource-group $rg_name --name $cluster_name --addons monitoring
```

### Download files your local machine to be able to drag&drop them to CloudShell window

[https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/deployworkspacetemplate.json](https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/deployworkspacetemplate.json)

[https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-deployment.yaml](https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-deployment.yaml)

[https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-ingress.yaml](https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-ingress.yaml)

[https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-service-cluster-ip.yaml](https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-service-cluster-ip.yaml)

[https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-service-lb.yaml](https://github.com/ezYakaEagle442/fr-cn-aks-workshop/blob/master/petclinic-service-lb.yaml)

Drag&drop the above files to CloudShell and check the files location in CloudShell
```sh
pwd
ls -al *.yaml
ls -al *.json
```

### Optionnal Play: Create Analytics Workspace


```sh
# https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli
# /!\ ATTENTION : check & modify location in the JSON template from https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli#create-and-deploy-template

# You can use `VIM <file you want to edit>` in Azure Cloud Shell to open the built-in text editor.
# You can upload files to the Azure Cloud Shell by dragging and dropping them
# You can also do a `curl -o filename.ext https://file-url/filename.ext` to download a file from the internet.

az group deployment create --resource-group $rg_name --template-file $analytics_workspace_template --name $analytics_workspace_name --parameters=workspaceName=$analytics_workspace_name

```

### Create Azure Container Registry
Note: Premium sku is a requirement to enable replication

```sh
az acr create --resource-group $rg_name --name $acr_registry_name --sku standard --location $location
# az acr create --resource-group $rg_name --name $acr_registry_name --sku Premium --location $location

# Get the ACR registry resource id
acr_registry_id=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "id" --output tsv)
echo "ACR registry ID :" $acr_registry_id
az acr repository list  --name $acr_registry_name # --resource-group $rg_name
az acr check-health --yes -n $acr_registry_name 

# Create role assignment
az role assignment create --assignee $sp_id --role acrpull --scope $acr_registry_id

docker_server="$(az acr show --name $acr_registry_name --resource-group $rg_name --query "name" --output tsv)"".azurecr.io"
echo "Docker server :" $docker_server

kubectl -n $target_namespace create secret docker-registry acr-auth \
        --docker-server="$docker_server" \
        --docker-username="$sp_id" \
        --docker-email="youremail@groland.grd" \
        --docker-password="$sp_password"

kubectl get secrets -n $target_namespace
```
### Optionnal Play: Design your Application for HA to support multiple  regions deployment & Enable geo-replication for container images
Note: Premium sku is a requirement to enable replication
```sh
# Configure https://docs.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication#configure-geo-replication
# https://docs.microsoft.com/en-us/cli/azure/acr/replication?view=azure-cli-latest
# location from az account list-locations : francecentral | northeurope | westeurope 
az acr replication create --location northeurope --registry $acr_registry_name --resource-group $rg_name
```

### Create Docker Image
```sh
# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-pack-build#example-build-java-image-with-heroku-builder
# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-quickstart-task-cli
git clone $git_url
cd spring-petclinic
# build app
mvn package

# On Azure Zulu JRE located at : /usr/lib/jvm/zulu-8-azure-amd64/
# to check which process runs eventually already on port 8080 :  netstat -anp | grep 8080 
# lsof -i :8080 | grep LISTEN
# ps -ef | grep PID

# Test the App
# mvn spring-boot:run

# https://docs.microsoft.com/en-us/java/azure/jdk/java-jdk-docker-images?view=azure-java-stable
# https://github.com/microsoft/java/blob/master/docker/alpine/Dockerfile.zulu-8u232-jre
# Java 8 image : mcr.microsoft.com/java/jdk:8u232-zulu-alpine
# Java 11 image :  mcr.microsoft.com/java/jre:11u5-zulu-alpine
artifact="spring-petclinic-2.2.0.BUILD-SNAPSHOT.jar"
echo -e "FROM mcr.microsoft.com/java/jre:11u5-zulu-alpine\n" \
"VOLUME /tmp \n" \
"ADD target/${artifact} app.jar \n" \
"RUN touch /app.jar \n" \
"EXPOSE 8080 \n" \
"ENTRYPOINT [ \""java\"", \""-Djava.security.egd=file:/dev/./urandom\"", \""-jar\"", \""/app.jar\"" ] \n"\
> Dockerfile

# other app snippet: https://github.com/microsoft/todo-app-java-on-azure/blob/master/deploy/aks/deployment.yml

az acr build -t "${docker_server}/spring-petclinic:{{.Run.ID}}" -r $acr_registry_name --resource-group $rg_name --file Dockerfile .
az acr repository list --name $acr_registry_name # --resource-group $rg_name
```

### Create Kubernetes deployment & Test container

<span style="color:red">/!\ IMPORTANT : the container image name is hardcoded and must be replaced, the Run ID was provided at the end of acr build command: 
${registryname}.azurecr.io/spring-petclinic:{{.Run.ID}}</span>

[https://docs.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos)

```sh
#az acr run -r $acr_registry_name --cmd "${docker_server}/spring-petclinic:dd2" /dev/null
kubectl apply -f petclinic-deployment.yaml -n $target_namespace
kubectl get deployments -n $target_namespace
kubectl get deployment petclinic -n $target_namespace 
kubectl get pods -l app=petclinic -o wide -n $target_namespace
kubectl get pods -l app=petclinic -o yaml -n $target_namespace | grep podIP

# check eventual errors:
k get events -n $target_namespace | grep -i "Error"

```
If there were no errors, you can skip the snippet below. In case of error, use the snippet below to found the error
```sh
for pod in $(k get po -n $target_namespace -o=name)
do
	k describe $pod | grep -i "Error"
	k logs $pod | grep -i "Error"
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/health
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/info
    # k exec $pod -n $target_namespace -it -- /bin/sh
done

# kubectl describe pod petclinic-YOUR-POD-ID -n $target_namespace
# kubectl logs petclinic-YOUR-POD-ID -n $target_namespace
# kubectl  exec -it "POD-UID" -n $target_namespace -- /bin/sh
```

### Create Kubernetes INTERNAL service

```sh
kubectl apply -f petclinic-service-cluster-ip.yaml -n $target_namespace
k get svc -n $target_namespace -o wide

# Use the command below to retrieve the Cluster-IP of the Service.
service_ip=$(kubectl get service petclinic-internal-service -n $target_namespace -o jsonpath="{.spec.clusterIP}")
kubectl get endpoints petclinic-lb-service -n $target_namespace
```

### HELM Setup
Helm version 3 does not come with any repositories predefined, so you’ll need [initialize the stable chart repository](https://v3.helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository)

```sh
kubectl create namespace ingress
helm version
helm get -h
# https://helm.sh/docs/intro/using_helm/
# You can see which repositories are configured using helm repo list
helm repo list

# Init default repo: https://hub.helm.sh/charts | https://kubernetes-charts.storage.googleapis.com/
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm search repo
helm search hub
helm search repo mongodb

helm repo update
# https://www.nginx.com/products/nginx/kubernetes-ingress-controller
helm install ingress stable/nginx-ingress --namespace ingress
helm upgrade --install ingress stable/nginx-ingress --namespace ingress
ing_ctl_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")

kubectl apply -f petclinic-ingress.yaml -n $target_namespace
kubectl get ingresses --all-namespaces
kubectl describe ingress petclinic -n $target_namespace

# All config proiperties ref: sur https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html 
echo "Your service is now exposed through an Ingress Controller at http://${ing_ctl_ip}"
echo "Check Live Probe with Spring Actuator : http://petclinic.${ing_ctl_ip}.nip.io/manage/health"
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/health" -i -X GET
echo ""
echo ""
# You should received an UP reply :
# {
#  "status" : "UP"
# }
echo "Check spring Management Info at http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET

```

### Create Kubernetes PUBLIC service (Load Balancer)
```sh
kubectl apply -f petclinic-service-lb.yaml -n $target_namespace
k get svc -n $target_namespace -o wide

# Standard load Balancer Use Case
# Use the command below to retrieve the External-IP of the Service. Make sure to allow a couple of minutes for the Azure Load Balancer to assign a public IP.
service_ip=$(kubectl get service petclinic-lb-service -n $target_namespace -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
# All config proiperties ref: sur https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html 
echo "Your service is now exposed through a Cluster IP at http://${service_ip}"
echo "Check Live Probe with Spring Actuator : http://${service_ip}/manage/health"
curl "http://${service_ip}/manage/health" -i -X GET
echo "\n"
# You should received an UP reply :
# {
#  "status" : "UP"
# }
echo "Check spring Management Info at http://${service_ip}/manage/info" -i -X GET
curl "http://${service_ip}/manage/info" -i -X GET
```

### Optionnal Play: Play with Kubernetes dashboard
```sh
# https://docs.microsoft.com/en-us/azure/aks/kubernetes-dashboard
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
az aks browse --resource-group $rg_name --name $cluster_name
```
### Leverage Kube-Advisor
```sh
# https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-resource-management#regularly-check-for-application-issues-with-kube-advisor
kubectl run --rm -i -t kube-advisor --image=mcr.microsoft.com/aks/kubeadvisor --restart=Never

kubectl run --rm -i -t kube-advisor --image=mcr.microsoft.com/aks/kubeadvisor --restart=Never --overrides="{ \"apiVersion\": \"v1\", \"spec\": { \"serviceAccountName\": \"kube-advisor\" } }"
```

### Leverage Kube-cost for cost analysis
```sh

```

### Leverage Kube-Hunter
```sh

```

### Leverage Kube-Bench
```sh

```
### Package your application with HELM
```sh

```



### Configure DNS

Free DNS: http://xip.io , https://nip.io, https://freedns.afraid.org, https://dyn.com/dns
```sh
$custom_dns
#service_ip=$(kubectl get service petclinic-lb-service -n $target_namespace -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
service_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
echo $service_ip

# https://docs.microsoft.com/en-us/cli/azure/network/public-ip?view=azure-cli-latest
# Get the resource-id of the public ip
managed_rg="mc_$rg_name"_"$cluster_name""_""$location"
echo $managed_rg

public_ip_id=$(az network public-ip list --subscription $subId --resource-group $managed_rg --query "[?ipAddress!=null]|[?contains(ipAddress, '$service_ip')].[id]" --output tsv)
echo $public_ip_id

In the Azure portal, go to All services / Public IP addresses / kubernetes-xxxx - Configuration ( the Ingress Controller IP) , then there is a field "DNS name label (optional)" ==> An "A record" that starts with the specified label and resolves to this public IP address will be registered with the Azure-provided DNS servers. Example: mylabel.westus.cloudapp.azure.com.

az network public-ip show --ids $public_ip_id --subscription $subId --resource-group $managed_rg

az network public-ip update --ids $public_ip_id --dns-name $dns_zone --subscription $subId --resource-group $managed_rg

#http://petclinic.francecentral.cloudapp.azure.com
#http://petclinic.internal.cloudapp.net

#http://petclinic.kissmyapp.francecentral.cloudapp.azure.com/ 
dns_zone="kissmyapp.francecentral.cloudapp.azure.com" 
az network dns zone create -g $rg_name -n $dns_zone
az network dns zone list -g $rg_name
az network dns record-set a add-record -g $rg_name -z $dns_zone -n www -a ${service_ip}
az network dns record-set list -g $rg_name -z $dns_zone

az network dns record-set cname create -g $rg_name -z $dns_zone -n petclinic-ingress
az network dns record-set cname set-record -g $rg_name -z $dns_zone -n petclinic-ingress -c www.$dns_zone
az network dns record-set cname show -g $rg_name -z $dns_zone -n petclinic-ingress
http://petclinic-ingress.kissmyapp.francecentral.cloudapp.azure.com/ 

```

### Monitoring
```sh

```
### Azure Arc
```sh
# https://github.com/Azure/azure-arc-kubernetes-preview
```

### Security

#### Define API server authorized IP ranges
```sh
# https://github.com/palma21/secureaks#define-api-server-authorized-ip-ranges

```


#### AKS private cluster 
```sh

```
#### Enabling Master logs - view Kubelet logs
```sh

```
#### Private  Application Gateway & Application Gateway Ingress Controller with private IP for Ingress endpoint internal routing

```sh
# https://github.com/palma21/secureaks#setup-app-gateway-ingress-controller

```
#### AKS Network Policies: Secure your Kubernetes workloads with virtual network and policy-driven communication paths between resources
```sh

```

#### Filter inbound traffic flows on API Server & Filter outbound traffic flows with Egress lockdown

##### Create Azure Firewall
```sh
# https://github.com/palma21/secureaks#creating-azure-firewall
```
#### Service-Mesh : Istio & Linkerd
```sh

```
#### Azure Policy with Open Policy Agent / Gatekeeper V3
```sh

```

#### AKS Periscope
```sh

```
#### Monitoring and Logging
```sh

```
#### SOC/SIEM integration
```sh

```


### Azure Security Center 

#### Container security in Security Center
```sh

```
#### Threat detection for Azure containers
```sh

```
#### Azure Kubernetes Service integration with Security Center
```sh

```
#### Azure Container Registry integration with Security Center
```sh

```

### IaC with terraform
```sh
# https://github.com/squasta/AzureKubernetesService-Terraform/blob/master/myconf.tfvars
# https://github.com/rhummelmose/aks-commander
```

###  Clean-Up
```sh
az aks delete --name $cluster_name --resource-group $rg_name
az acr delete --resource-group $rg_name --name $acr_registry_name
az keyvault delete --location $location --name $vault_name --resource-group $rg_name
az network vnet delete --name $vnet_name --resource-group $rg_name --location $location
az network vnet subnet delete --name $subnet_name --vnet-name $vnet_name --resource-group $rg_name 
az network route-table route delete -g  $rg_name --route-table-name $route_table -n $route
az network route-table delete -g  $rg_name -n $route_table
az network dns record-set a delete -g $rg_name -z $dns_zone -n www 
az network dns zone delete -g $rg_name -n $dns_zone
az network dns zone list -g $rg_name
az group delete --name $rg_name
```

<span style="color:red">**/!\ IMPORTANT** </span> : Decide to keep or delete your service principal
```sh
az ad sp delete --id $sp_id
rm spp.txt
rm spid.txt
```