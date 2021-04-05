# AzureCli Commands

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
