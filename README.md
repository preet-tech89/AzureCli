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

New-AzVm `
    -ResourceGroupName "testrg" `
    -Name "winvmshell" `
    -Location "East US" `
    -VirtualNetworkName "winvmshellvnet" `
    -SubnetName "winvmshellsubnet" `
    -SecurityGroupName "winvmshellnsg" `
    -PublicIpAddressName "myPublicIpAddress" `
    -OpenPorts 80,3389

#List ip

Get-AzPublicIpAddress -ResourceGroupName "testrg" | Select "IpAddress"

#Cleamup

Remove-AzResourceGroup -Name testrg
