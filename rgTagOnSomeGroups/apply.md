# Create policy and assing to just one resource group
az policy definition create -n tagPolicy \
    --display-name "tagPolicy" \
    --mode all \
    --rules taggingSomeRgs.rules.json

az policy assignment create -n tagPolicy \
    --display-name "tagPolicy" \
    --policy tagPolicy \
    --sku standard

# Create untagged resource on RG named tomas-rg (should not fail by policy)
az group create -l westeurope -n tomas-rg

# Create untagged resource on RG named tomas-rg (should fail by policy)
az group create -l westeurope -n martin-rg

# Clean up
az group delete -n tomas-rg -y --no-wait
az group delete -n martin-rg -y --no-wait
az policy assignment delete -n tagPolicy
az policy definition delete -n tagPolicy