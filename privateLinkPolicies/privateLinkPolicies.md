**WORK IN PROGRESS**

## Close public endpoint access on PaaS
In enterprise scenarios use of Private Link might be declared mandatory nevertheless configuration of private link does not automatically disable access via public endpoint. While some wizards can automate this, when manual private link configuration, ARM templates or CLI scripts are used, administrator can easily forget to disable access over public endpoints. This policy will configure required settings automatically.

```bash
az policy definition create -n defaultDisableAzureSqlPublic \
    --display-name "If not specified disable public access on Azure SQL" \
    --mode all \
    --rules defaultDisableAzureSqlPublic.rules.json

az policy definition create -n blockAzureSqlPublic \
    --display-name "Block attempt to enable public access on Azure SQL" \
    --mode all \
    --rules blockAzureSqlPublic.rules.json
```


```bash
az group create -n policy -l westeurope

az policy assignment create -n defaultDisableAzureSqlPublic-rg-policy \
    --display-name "If not specified disable public access on Azure SQL" \
    --policy defaultDisableAzureSqlPublic \
    --resource-group policy \
    --sku standard

az policy assignment create -n blockAzureSqlPublic-rg-policy \
    --display-name "Block attempt to enable public access on Azure SQL" \
    --policy blockAzureSqlPublic \
    --resource-group policy \
    --sku standard
```

```bash
az sql server create -l westeurope -g policy -n tomas-sql-server0239 -u tomas -p Azure12345678
az sql server create -l westeurope -g policy -n tomas-sql-server0237 -u tomas -p Azure12345678 -e true
az sql server create -l westeurope -g policy -n tomas-sql-server0238 -u tomas -p Azure12345678 -e false
```

az policy assignment delete -n defaultDisableAzureSqlPublic-rg-policy --resource-group policy
az policy assignment delete -n defaultDisableAzureSqlPublic-rg-policy --resource-group policy
az policy definition delete -n blockAzureSqlPublic
az policy definition delete -n blockAzureSqlPublic

az group delete -n policy -y --no-wait