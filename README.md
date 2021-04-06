# AzureCli and poweshell Commands

#Login

az login
az account-set --subscription "Pay-As-You-Go"

#list

az group list --output table

#Create RG

az group create \
    --name "testrg" \
    --location "centralus"
    
#Create win VM

az vm create \
  --resource-group "testrg" \
  --name "winclidemo" \
  --image "win2019datacenter" \
  --admin-username "demo" \
  --admin-password "" 
  
#Create linux VM

az vm create \
  --resource-group "testrg" \
  --name "linuxclidemo" \
  --image "UbuntuLTS" \
  --admin-username "demo" \
  --authentication-type "SSH" \
  --SSH-key-value ~/.ssh/id_ssh.pub
  
#Open port win

az vm open-port \
  --resource-group "testrg" \
  --name "winclidemo" \
  --port "3389"
  
#Open port linux

az vm open-port \
  --resource-group "testrg" \
  --name "linuxclidemo" \
  --port "22"
  
#fetch public ip address

az vm list-ip-addresses \
  --resource-group "testrg" \
  --name "linuxclidemo" \
  --output table
  
#Login to win vm

In Powershell: mstsc /v:publicIpAddress

#Login to linux vm

ssh username@public_ip_address

#CLeanup

az group delete --name "testrg" 

#Powershell Command

#Login

Connect-AzAccount -SubscriptionName "Pay-As-You-Go"

#Point to correct context

Set-AzContext -SubscriptionName "Pay-As-You-Go"

#RG creation

New-AzResourceGroup -Name testrg -Location EastUS

#VM

#Create a credential to use in the VM creation

$username = 'demoadmin'
$password = ConvertTo-SecureString '<password_tobe>' -AsPlainText -Force
$WindowsCred = New-Object System.Management.Automation.PSCredential ($username, $password)

New-AzVM `
    -ResourceGroupName 'testrg' `
    -Name 'gpdemo-win' `
    -Image 'Win2019Datacenter' `
    -Credential $WindowsCred `
    -OpenPorts 3389 ##RDP port

#List ip

Get-AzPublicIpAddress -ResourceGroupName "testrg" | Select "IpAddress"

#Cleamup

Remove-AzResourceGroup -Name testrg


#CONTAINERS

#Dockerfile example:

FROM node

RUN mkdir /app
WORKDIR /app

COPY ./app/ ./
COPY ./config.sh ./

RUN bash config.sh #in case any custom script to be executed

EXPOSE 80
ENTRYPOINT ["node", "app.js"]

#Install docker  https://docs.docker.com/desktop/#download-and-install

#Build the container and tag it, defined in Dockerfile (dot at the end specify location of file)

docker build -t myapp:v1 .

#Run the container locally

docker run --name myapp --publish 8080:80 --detach myapp:v1 # --name specifies container name

curl http://localhost:8080

#Delete the running container

docker stop myapp

docker rm myapp

docker image ls myapp:v1

#remove image 

docker image rm myapp or docker rmi myapp

#Create Azure Container Registry and Push

az login

az account set --subscription "Pay-As-You-Go"

#A) - Create Resource Group

az group create \
    --name testrg \
    --location centralus

#B) - Create Azure Container Registry
#SKUs parameter include Basic, Standard and Premium (speed, replication, adv security features)
#https://docs.microsoft.com/en-us/azure/container-registry/container-registry-skus#sku-features-and-limits

ACR_NAME='gpdemoacr'  #Needs to be globally unique inside of Azure, bash global variable to store

az acr create \
    --resource-group testrg \
    --name $ACR_NAME \
    --sku Standard

#C) - Log into ACR to push containers, will use current azure CLI context
#Get the loginServer, used in the image tag

ACR_LOGINSERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)

echo $ACR_LOGINSERVER

# Tag the container image using the login server name
#[loginUrl]/[repository:][tag] Format

docker tag myapp:v1 $ACR_LOGINSERVER/myapp:v1

docker image ls $ACR_LOGINSERVER/myapp:v1

docker image ls

#D) - Push image to Azure Container Registry

docker push $ACR_LOGINSERVER/myapp:v1


#E) - Get list of repositories and images/tags in our Azure Container Registry

az acr repository list --name $ACR_NAME --output table

az acr repository show-tags --name $ACR_NAME --repository myapp --output table

#If we do not want to build locally then push, then we can build in ACR with the help of Tasks.

#A) - use ACR build to build our image in azure and then push that into ACR

az acr build --image "myapp:v1-acr-task" --registry $ACR_NAME . ## Dot at the end specific location of Dockerfile


#B) Two images are there, one built locally and one with ACR Tasks

az acr repository show-tags --name $ACR_NAME --repository myapp --output table

az acr login --name $ACR_NAME

####Running Container in Azure Container Instance (PaaS)

#A) - Deploy container from the public registry like DockerHub.

az container create \
    --resource-group testrg \
    --name gpdemo-node-cli \
    --dns-name-label gpdemo-node-cli \  #gpdemo-node-cli.<region>.azurecontainer.io
    --image docker.io/library/node \
    --ports 80


#Show container info

az container show --resource-group 'testrg' --name 'gpdemo-node-cli' 


#Retrieve URL, the format is [name].[region].azurecontainer.io

URL=$(az container show --resource-group 'testrg' --name 'gpdemo-node-cli' --query ipAddress.fqdn | tr -d '"') 

echo "http://$URL"

#B) - Deploy the container from ACR (Private Repository), using service principal as authentication

ACR_NAME='gpdemoacr'  #Needs to be globally unique inside of Azure, bash global variable to store

#B.1) - Get the registry ID and login server, that will use in the security

ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

ACR_LOGINSERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)

echo "ACR ID: $ACR_REGISTRY_ID"

echo "ACR Login Server: $ACR_LOGINSERVER"


#B.2 - Create a service principal and get the password and ID, this will allow Azure Container Instances to Pull Images from our Azure Container Registry
#SP_NAME=acr-service-principal

SP_PASSWD=$(az ad sp create-for-rbac \
    --name http://$ACR_NAME-pull \  ### RBAC service name
    --scopes $ACR_REGISTRY_ID \ ##Scope to specific ACR
    --role acrpull \ ##Least role to pull image
    --query password \  ## Get password for creation container
    --output tsv)

SP_APPID=$(az ad sp show \
    --id http://$ACR_NAME-pull \
    --query appId \
    --output tsv)

echo "Service principal ID: $SP_APPID"

echo "Service principal password: $SP_PASSWD"


#B.3 - Create the container in ACI, this will pull our image named from provate ACR
#$ACR_LOGINSERVER is gpdemoacr.azurecontainer.io.

az container create \
    --resource-group testrg \
    --name gpdemo-node-cli \
    --dns-name-label gpdemo-node-cli \
    --ports 80 \
    --image $ACR_LOGINSERVER/myapp:v1 \
    --registry-login-server $ACR_LOGINSERVER \
    --registry-username $SP_APPID \
    --registry-password $SP_PASSWD 


#B.4 - Confirm the container is running and test access to application, i.e instanceView.state in the response

az container show --resource-group testrg --name gpdemo-node-cli


#Get the URL of the container running in ACI...

URL=$(az container show --resource-group testrg --name gpdemo-node-cli --query ipAddress.fqdn | tr -d '"') 
echo $URL
curl $URL


# Pull the logs from the container

az container logs --resource-group testrg --name gpdemo-node-cli

#Delete the running container

az container delete  \
    --resource-group testrg \
    --name gpdemo-node-cli \
    --yes

#Clean up , this will delete all of the ACIs and the ACR deployed in this resource group.
#Delete the local container images

az group delete --name testrg --yes

docker image rm gpdemoacr.azurecr.io/myapp:v1

docker image rm myapp:v1
