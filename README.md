# nvidia-app-mod

## Lab 1

Create a new resource group and use it for your labs.

```powershell
```

## Lab 2

Connect to remote VM using password

```bash
ssh azureuser@172.212.82.21
```

```bash
sudo gpasswd -a $USER docker
exit
```

Run below commands to setup NVIDIA model

```bash
export NGC_API_KEY="nvapi-92kPYcNVki2yYXCxxxxx" #edit
echo "export NGC_API_KEY=your-ngc-api-key" >> ~/.bashrc
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

```bash
export NIM_TAGS_SELECTOR="name=parakeet-1-1b-ctc-riva-en-us,mode=all"
docker run -it --rm --name=riva-asr \
   --gpus '"device=0"' \
   --shm-size=8GB \
   -e NGC_API_KEY \
   -e NIM_HTTP_API_PORT=9000 \
   -e NIM_GRPC_API_PORT=50051 \
   -p 9000:9000 \
   -p 50051:50051 \
   -e NIM_TAGS_SELECTOR \
   nvcr.io/nim/nvidia/riva-asr:1.3.0
```

## Lab 3

Build monolithic app

```powershell
docker build -t "pycontosohotel:v1.0.0" .
```

```powershell
az login --use-device-code
```

```powershell
$ACR_NAME = "acrxxx" #edit
az acr login --name $ACR_NAME
docker tag "pycontosohotel:v1.0.0" "$ACR_NAME.azurecr.io/pycontosohotel:v1.0.0"
docker push "$ACR_NAME.azurecr.io/pycontosohotel:v1.0.0"
```

Deploy database and connect with app locally

```powershell
Connect-AzAccount
```

```powershell
$RG = "rg-daniel" #edit
.\iac\manageIac.ps1 -iacAction create -passwd "1234ABcd!" -deploy "postgresql" -rgname $RG -location "eastus2"
```

```powershell
docker run -p 8000:8000 -e POSTGRES_CONNECTION_STRING="host=<connstr>" pycontosohotel:v1.0.0
```

POSTGRES_CONNECTION_STRING example: j3ufxxxx.postgres.database.azure.com;port=5432;database=pycontosohotel;user=contosoadmin;password=1234ABcd!;

## Lab 4

App modernisation, split Frontend and Backend apps

```powershell
cd C:\temp\
mkdir -p ContosoHotel/UpdatedApp/Frontend 
mkdir -p ContosoHotel/UpdatedApp/Backend
```

```powershell
-- run in powershell
cp startup.* C:\temp\ContosoHotel\UpdatedApp\Frontend
cp uwsgi.ini C:\temp\ContosoHotel\UpdatedApp\Frontend
cp Dockerfile C:\temp\ContosoHotel\UpdatedApp\Frontend
cp *.docker* C:\temp\ContosoHotel\UpdatedApp\Frontend
cp requirements.txt C:\temp\ContosoHotel\UpdatedApp\Frontend

cp -r C:/temp/ContosoHotel/contoso_hotel/static C:\temp\ContosoHotel\UpdatedApp\Frontend\contoso_hotel\static
cp -r C:/temp/ContosoHotel/contoso_hotel/templates C:\temp\ContosoHotel\UpdatedApp\Frontend\contoso_hotel\
cp C:/temp/ContosoHotel/contoso_hotel/*.py C:\temp\ContosoHotel\UpdatedApp\Frontend\contoso_hotel\

cp *.sql C:\temp\ContosoHotel\UpdatedApp\Backend
cp startup.* C:\temp\ContosoHotel\UpdatedApp\Backend
cp uwsgi.ini C:\temp\ContosoHotel\UpdatedApp\Backend
cp *docker* C:\temp\ContosoHotel\UpdatedApp\Backend
cp requirements.txt C:\temp\ContosoHotel\UpdatedApp\Backend

cp -r C:/temp/ContosoHotel/contoso_hotel/dblayer C:\temp\ContosoHotel\UpdatedApp\Backend\contoso_hotel\dblayer
cp C:/temp/ContosoHotel/contoso_hotel/*.py C:\temp\ContosoHotel\UpdatedApp\Backend\contoso_hotel\
```

Build frontend and backend containers

```powershell
$ACR_NAME = "acrxxx" #edit
cd C:\temp\ContosoHotel\UpdatedApp\Frontend
docker build -t "pycontosohotel-frontend:v1.0.0" .
docker tag "pycontosohotel-frontend:v1.0.0" "$ACR_NAME.azurecr.io/pycontosohotel-frontend:v1.0.0"
docker push "$ACR_NAME.azurecr.io/pycontosohotel-frontend:v1.0.0"
```

```powershell
cd C:\temp\ContosoHotel\UpdatedApp\Backend
docker build -t "pycontosohotel-backend:v1.0.0" .
docker tag "pycontosohotel-backend:v1.0.0" "$ACR_NAME.azurecr.io/pycontosohotel-backend:v1.0.0"
docker push "$ACR_NAME.azurecr.io/pycontosohotel-backend:v1.0.0"
```

Deploy containers to Container Apps

```powershell
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

```powershell
$RG = "rg-daniel" #edit
$AZURE_REGION = "eastus2" #edit
$ACR_NAME = "acrxxx" #edit

$CONTOSO_HOTEL_ENV = "contosoenv$(Get-Random -Minimum 100000 -Maximum 999999)"
Write-Host -ForegroundColor Green  "CONTOSO_HOTEL_ENV is: $CONTOSO_HOTEL_ENV"
```

```powershell
az acr update -n $ACR_NAME --admin-enabled true
$CONTOSO_ACR_CREDENTIAL = az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv
```

Create Container App env

```powershell
az containerapp env create --name "$CONTOSO_HOTEL_ENV" --resource-group $RG --location "$AZURE_REGION"
Write-Host -ForegroundColor Green  "Default Domain is: $(az containerapp env show --name "$CONTOSO_HOTEL_ENV" --resource-group $RG --query "properties.defaultDomain" -o tsv)"
```

Deploy backend app

```powershell
az containerapp create --name "backend" --resource-group $RG --environment "$CONTOSO_HOTEL_ENV" --image "$ACR_NAME.azurecr.io/pycontosohotel-backend:v1.0.0" --target-port 8000 --ingress external --transport http --registry-server "$ACR_NAME.azurecr.io" --registry-username "$ACR_NAME" --registry-password "$CONTOSO_ACR_CREDENTIAL" --env-vars POSTGRES_CONNECTION_STRING="host=<connstr>;"

$CONTOSO_BACKEND_URL = "https://$(az containerapp show --name "backend" --resource-group $RG --query 'properties.configuration.ingress.fqdn' -o tsv)"
Write-Host -ForegroundColor Green  "Backend URL is: $CONTOSO_BACKEND_URL"
```

POSTGRES_CONNECTION_STRING example: j3ufxxxx.postgres.database.azure.com;port=5432;database=pycontosohotel;user=contosoadmin;password=1234ABcd!;

Deploy frontend app

```powershell
az containerapp create --name "frontend" --resource-group $RG --environment "$CONTOSO_HOTEL_ENV" --image "$ACR_NAME.azurecr.io/pycontosohotel-frontend:v1.0.0" --target-port 8000 --ingress external --transport http --registry-server "$ACR_NAME.azurecr.io" --registry-username "$ACR_NAME" --registry-password "$CONTOSO_ACR_CREDENTIAL" --env-vars "API_BASEURL=$CONTOSO_BACKEND_URL"

$CONTOSO_FRONTEND_URL = "https://$(az containerapp show --name "frontend" --resource-group $RG --query 'properties.configuration.ingress.fqdn' -o tsv)"
Write-Host -ForegroundColor Green  "Frontend URL is: $CONTOSO_FRONTEND_URL"
```

```powershell
az containerapp ingress cors update --name "backend" --resource-group $RG --allowed-origins * --allowed-methods ***
```

## Lab 5

```powershell
git clone https://github.com/microsoft/TechExcel-Modernize-applications-to-be-AI-ready.git AssetsRepo
```

Create and upload files to storage

```powershell
$RG = "rg-daniel"
$AZURE_REGION = "eastus2"
$CONTOSO_STORAGE_ACCOUNT_NAME="contososa$(Get-Random -Minimum 100000 -Maximum 999999)"
Write-Host -ForegroundColor Green  "Storage account name is: " $CONTOSO_STORAGE_ACCOUNT_NAME

az storage account create --name $CONTOSO_STORAGE_ACCOUNT_NAME --resource-group $RG --location $AZURE_REGION --sku Standard_LRS
az storage container create --name brochures --account-name $CONTOSO_STORAGE_ACCOUNT_NAME
```

```powershell
az storage blob upload-batch --account-name $CONTOSO_STORAGE_ACCOUNT_NAME --destination brochures --source "Assets\PDFs" --pattern "*.pdf" --overwrite
```

Create AI Search

```powershell
$CONTOSO_SEARCH_SERVICE_NAME="contososrch$(Get-Random -Minimum 100000 -Maximum 999999)"
az search service create --name $CONTOSO_SEARCH_SERVICE_NAME --resource-group $RG --sku Basic --location $AZURE_REGION  --auth-options aadOrApiKey --aad-auth-failure-mode http403 --identity-type SystemAssigned
```

Configure RBAC permissions

```powershell
$CONTOSO_SEARCH_SERVICE_NAME="contososrchxxxx" #edit
$CONTOSO_STORAGE_ACCOUNT_NAME="contososaxxxx" #edit
$CONTOSO_OPENAI_NAME="aoaixxxx" #edit
```

```powershell
$SEARCH_IDENTITY=$(az search service show --name $CONTOSO_SEARCH_SERVICE_NAME --resource-group $RG --query identity.principalId -o tsv)
$AI_IDENTITY=$(az cognitiveservices account identity show --name $CONTOSO_OPENAI_NAME --resource-group $RG --query principalId -o tsv)
```

```powershell
$STORAGE_SCOPE=$(az storage account show --name $CONTOSO_STORAGE_ACCOUNT_NAME --resource-group $RG --query id -o tsv)
az role assignment create --role "Storage Blob Data Contributor" --assignee $SEARCH_IDENTITY --scope $STORAGE_SCOPE
az role assignment create --role "Storage Blob Data Contributor" --assignee $AI_IDENTITY --scope $STORAGE_SCOPE
```

```powershell
$AI_SCOPE=$(az cognitiveservices account show --name $CONTOSO_OPENAI_NAME --resource-group $RG --query id -o tsv)
az role assignment create --role "Cognitive Services OpenAI Contributor" --assignee $SEARCH_IDENTITY --scope $AI_SCOPE
```

```powershell
$SEARCH_SCOPE=$(az search service show --name $CONTOSO_SEARCH_SERVICE_NAME --resource-group $RG --query id -o tsv)
az role assignment create --role "Search Index Data Contributor" --assignee $AI_IDENTITY --scope $SEARCH_SCOPE
az role assignment create --role "Search Index Data Reader" --assignee $AI_IDENTITY --scope $SEARCH_SCOPE
az role assignment create --role "Search Service Contributor" --assignee $AI_IDENTITY --scope $SEARCH_SCOPE
```


## Lab 6

Connect to postgres sql

```powershell
export PGHOST="j3ufxxxx.postgres.database.azure.com" #edit
export PGUSER="contosoadmin"
export PGPORT="5432"
export PGDATABASE="pycontosohotel"
export PGPASSWORD="1234ABcd!"
psql
```

SQL Query to create user

```powershell
CREATE USER promptflow WITH PASSWORD '1234ABCD!';

GRANT SELECT ON TABLE hotels TO promptflow;
GRANT SELECT ON TABLE bookings TO promptflow;
GRANT SELECT ON TABLE visitors TO promptflow;
GRANT EXECUTE ON FUNCTION getroomsusagewithintimespan TO promptflow;
```

Create AI Hub

```powershell
$AI_HUB_NAME="ai-hub$(Get-Random -Minimum 100000 -Maximum 999999)"
Write-Host -ForegroundColor Green  "AI Hub name is: " $AI_HUB_NAME
```

Create Prompt Flow configurations

```powershell
hostname: j3ufxxxx.postgres.database.azure.com #edit
user: promptflow
port:5432
database: pycontosohotel
passwd: 1234ABCD! (secret)
conn name: postgresql
```

```powershell
az network nsg rule create --resource-group $RG --nsg-name vmi3222-nsg --name allow-http --protocol tcp --priority 100 --destination-port-range 9000
az network nsg rule create --resource-group $RG --nsg-name vmi3222-nsg --name allow-grpc --protocol tcp --priority 110 --destination-port-range 50051

curl http://172.212.82.21:9000/v1/health/ready
```

Setup Speech App

```powershell
git clone https://github.com/CloudLabsAI-Azure/NVIDIA-Speech-to-text.git
```

NVIDIA model & Azure AI Foundry configuration

```powershell
RIVA_SERVER="172.212.xx.xx:50051" #edit
LANGUAGE_CODE=en-US
PORT=5000
AZURE_ENDPOINT_URL="https://contosopf-xxxx.eastus2.inference.ml.azure.com/score" #edit
AZURE_API_KEY="7zAw45xtgEBxxxx" #edit
```

Run Speech App locally

```powershell
pip install -r requirements
python app.py
```

```powershell
$RG = "rg-daniel"
$AZURE_REGION = "eastus2" #edit
$ACR_NAME = "acrxxx" #edit
$CONTOSO_HOTEL_ENV = "contosoenv996543" #edit
$CONTOSO_ACR_CREDENTIAL = az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv
```

Build Speech App

```powershell
docker build -t "chatapp:v1.0.0" .
docker tag "chatapp:v1.0.0" "$ACR_NAME.azurecr.io/chatapp:v1.0.0"
docker push "$ACR_NAME.azurecr.io/chatapp:v1.0.0"
```

Deploy Speech app

```powershell
$chatapp = "chatapp$(Get-Random -Minimum 100000 -Maximum 999999)"

az containerapp create --name "$chatapp" --resource-group $RG --environment "$CONTOSO_HOTEL_ENV" --image "$ACR_NAME.azurecr.io/chatapp:v1.0.0" --target-port 5000 --ingress external --transport http --registry-server "$ACR_NAME.azurecr.io" --registry-username "$ACR_NAME" --registry-password "$CONTOSO_ACR_CREDENTIAL"
$CONTOSO_CHAT_URL = "https://$(az containerapp show --name "$chatapp" --resource-group $RG --query 'properties.configuration.ingress.fqdn' -o tsv)"
Write-Host -ForegroundColor Green  "Chatapp URL is: $CONTOSO_CHAT_URL"
```


## Lab 7

App Setting Configuration

```powershell
AZURE_OPENAI_ENDPOINT="https://aoaixxxx.openai.azure.com/" #edit
AZURE_OPENAI_API_KEY="ATNNceXifFgxxxx" #edit
AZURE_OPENAI_DEPLOYMENT_ID="gpt-4o"
AZURE_AI_SEARCH_ENDPOINT="https://contososrchxxxx.search.windows.net" #edit
AZURE_AI_SEARCH_INDEX="brochures-vector" #edit
AZURE_AI_SEARCH_API_KEY="jBfoENDvKzPvof9xxxx" #edit
PGHOST="j3ufxxxx.postgres.database.azure.com" #edit
PGPORT="5432"
PGUSER="promptflow"
PGDATABASE="pycontosohotel"
PGPASSWORD="1234ABCD!"
```

```powershell
get-content .env | foreach {
$name, $value = $_.split('=')
set-content env:\$name $value
}
```

Create Prompt Flow connections

```powershell
pf connection create --file azure_openai.yaml --name azure_openai --set "api_base=$env:AZURE_OPENAI_ENDPOINT" --set "api_key=$env:AZURE_OPENAI_API_KEY"
pf connection create --file azure_ai_search.yaml --name azure_ai_search --set "api_base=$env:AZURE_AI_SEARCH_ENDPOINT" --set "api_key=$env:AZURE_AI_SEARCH_API_KEY"
pf connection create --file postgresql.yaml --name postgresql --set "configs.hostname=$env:PGHOST" --set "configs.port=$env:PGPORT" --set "configs.user=$env:PGUSER" --set "configs.database=$env:PGDATABASE" --set "secrets.passwd=$env:PGPASSWORD"

pf connection list | ConvertFrom-Json | Select-Object name, type |Format-Table
```

Run Prompt Flow locally

```powershell
pip install -r requirements.txt
pf flow test --flow . --interactive
```

```powershell
$ACR_NAME = "acrxxx" #edit
az acr login --name "$ACR_NAME"
```

Build Prompt Flow as Chatbot app

```powershell
pf flow build --source . --output docker-dist --format docker
Copy-Item -Path .\azure_openai.yaml -Destination .\docker-dist\connections -Force
Copy-Item -Path .\azure_ai_search.yaml -Destination .\docker-dist\connections -Force
Copy-Item -Path .\postgresql.yaml -Destination .\docker-dist\connections -Force
```

```powershell
docker build -t "$ACR_NAME.azurecr.io/chatbot:v1.0.0" ./docker-dist
docker push "$ACR_NAME.azurecr.io/chatbot:v1.0.0"
Get-ChildItem -Path '.\docker-dist\connections' -Filter '*.yaml' | Get-Content | Select-String 'env:'
# Remove-Item -Recurse -Force ./docker-dist
```

Configure Chatbot app

```powershell
$CONTOSO_HOTEL_ENV = "contosoenv996543" #edit
$AZURE_OPENAI_ENDPOINT = "https://aoaixxxx.openai.azure.com/" #edit
$AZURE_OPENAI_API_KEY = "ATNNceXifFgxxxx" #edit
$AZURE_AI_SEARCH_ENDPOINT = "https://contososrchxxxx.search.windows.net" #edit
$AZURE_AI_SEARCH_API_KEY = "jBfoENDvKzPvof9xxxx" #edit
$PGHOST = "j3ufxxxx.postgres.database.azure.com" #edit
```

```powershell
$RG = "rg-daniel" #edit
$CONTOSO_ACR_CREDENTIAL = az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv
$PGUSER = "promptflow"
$PGPASSWORD = "1234ABCD!"
$PGDATABASE = "pycontosohotel"
$PGPORT = "5432"
```

Deploy Chatbot app

```powershell
az containerapp create --name "chatbot" --resource-group $RG --environment "$CONTOSO_HOTEL_ENV" `
--image "$ACR_NAME.azurecr.io/chatbot:v1.0.0" --target-port 8080 --transport http `
--registry-server "$ACR_NAME.azurecr.io" --registry-username "$ACR_NAME" --registry-password "$CONTOSO_ACR_CREDENTIAL" `
--secrets "searchkey=$AZURE_AI_SEARCH_API_KEY" "openaikey=$AZURE_OPENAI_API_KEY" "pgpassword=$PGPASSWORD" `
--env-vars "AZURE_AI_SEARCH_ENDPOINT=$AZURE_AI_SEARCH_ENDPOINT" "AZURE_AI_SEARCH_API_KEY=secretref:searchkey" `
"AZURE_OPENAI_ENDPOINT=$AZURE_OPENAI_ENDPOINT" "AZURE_OPENAI_API_KEY=secretref:openaikey" `
"PGHOST=$PGHOST" "PGPORT=$PGPORT" "PGUSER=$PGUSER" "PGDATABASE=$PGDATABASE" "PGPASSWORD=secretref:pgpassword" --ingress external
$CONTOSO_CHATBOT_URL = "https://$(az containerapp show --name "chatbot" --resource-group $RG --query 'properties.configuration.ingress.fqdn' -o tsv)"
Write-Host -ForegroundColor Green  "Promptflow URL is: $CONTOSO_CHATBOT_URL"
```

Configure Frontend & Backend app

```powershell
$CONTOSO_BACKEND_URL = "https://$(az containerapp show --name "backend" --resource-group $RG --query 'properties.configuration.ingress.fqdn' -o tsv)"
az containerapp update --name "frontend" --resource-group $RG --set-env-vars "CHATBOT_BASEURL=$CONTOSO_BACKEND_URL"
az containerapp update --name "backend" --resource-group $RG --set-env-vars "CHATBOT_BASEURL=$CONTOSO_CHATBOT_URL"
```

