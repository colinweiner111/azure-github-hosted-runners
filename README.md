# GitHub-Hosted Runners — Azure Private Networking
## Deployment Checklist | Existing VNet, New Subnet

**About doc:** https://docs.github.com/en/enterprise-cloud@latest/admin/configuring-settings/configuring-private-networking-for-hosted-compute-products/about-azure-private-networking-for-github-hosted-runners-in-your-enterprise

**Config doc:** https://docs.github.com/en/enterprise-cloud@latest/admin/configuring-settings/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise

---

## Pre-Flight Checklist

### Azure
- [ ] Subscription Contributor + Network Contributor roles on the target subscription
- [ ] NetworkSettings resource will be created in the **same subscription** as the existing VNet
- [ ] All resources (NSG, subnet, NetworkSettings) will be deployed in the **same Azure region** as the existing VNet
- [ ] Entra admin available to consent to two enterprise app registrations:
  - GitHub CPS Network Service — `85c49807-809d-4249-86e7-192762525474`
  - GitHub Actions API — `4435c199-c3da-46b9-a61d-76de3f2c9f82`
- [ ] Target region is in GitHub's supported region list (see About doc)
- [ ] Create a new dedicated subnet in the existing VNet — recommended over using an existing subnet:
  - Documented path — using an existing subnet is not explicitly covered in GitHub docs
  - Clean isolation — runner NSG rules don't impact other workloads
  - Easier to tear down — no other workloads depending on it
  - **Note:** If reusing an existing subnet, ensure all existing NICs are removed before delegation — the GitHub.Network service association link will fail otherwise
- [ ] Subnet sized correctly (GitHub recommends a **30% buffer** over max concurrency):

| Max Concurrent Jobs | +30% Buffer | Min Subnet Size | Usable IPs |
|---|---|---|---|
| Up to 10 | 13 | /27 | 27 |
| Up to 50 | 65 | /25 | 123 |
| Up to 100 | 130 | /24 | 251 |
| Up to 300 | 390 | /23 | 507 |

- [ ] No TLS inspection on outbound traffic from the subnet (if TLS interception is required, intermediate certificates can be installed via a [custom image](https://docs.github.com/en/enterprise-cloud@latest/actions/how-tos/manage-runners/larger-runners/use-custom-images))
- [ ] Private endpoints deployed for target services (ACR, Key Vault, Storage, etc.)
- [ ] Private DNS Zones linked to the VNet — runners must resolve private endpoint FQDNs

### GitHub
- [ ] GitHub Enterprise Cloud subscription confirmed
- [ ] Enterprise owner role on GitHub — person in the room?
- [ ] GitHub token with `read:enterprise` scope — needed to retrieve databaseId
- [ ] Larger runners available (2–64 vCPU Ubuntu or Windows)

> ⚠️ **Read the About doc with your security and Entra teams before running any scripts** — it has the full RBAC permissions list and app registration details that need sign-off.

---

## Phase 1 — Azure Setup

> ⚠️ **Do NOT run the GitHub provided bash script as-is** — it creates a new resource group and VNet which we don't need. We are using an existing VNet. Run the individual commands below instead, which skip those steps.

### Prepare

1. Get your GitHub Enterprise `databaseId`:
    ```bash
    curl -H "Authorization: Bearer TOKEN" -X POST \
      -d '{ "query": "query($slug: String!) { enterprise (slug: $slug) { slug databaseId } }", "variables": { "slug": "ENTERPRISE_SLUG" } }' \
      https://api.github.com/graphql
    ```

2. Save the `actions-nsg-deployment.bicep` file — copy the Bicep content from the config doc and save it locally in the directory you will run the commands from

3. Note the `databaseId` value — used as `DATABASE_ID` in the commands below

### Azure CLI Commands — Run in Order

1. Login and set subscription
    ```bash
    az login
    az account set --subscription $SUBSCRIPTION_ID
    ```

2. Register GitHub.Network resource provider
    ```bash
    az provider register --namespace GitHub.Network
    ```

3. Deploy NSG rules
    ```bash
    az deployment group create \
      --resource-group $RESOURCE_GROUP_NAME \
      --template-file ./actions-nsg-deployment.bicep \
      --parameters location=$AZURE_LOCATION nsgName=$NSG_NAME
    ```

4. Create new dedicated subnet in the existing VNet
    ```bash
    az network vnet subnet create \
      --resource-group $RESOURCE_GROUP_NAME \
      --vnet-name $VNET_NAME \
      --name $SUBNET_NAME \
      --address-prefixes $SUBNET_PREFIX
    ```

5. Delegate subnet to `GitHub.Network/networkSettings` and associate NSG (single command per official docs)
    ```bash
    az network vnet subnet update \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $SUBNET_NAME \
      --vnet-name $VNET_NAME \
      --delegations GitHub.Network/networkSettings \
      --network-security-group $NSG_NAME
    ```

6. Create the NetworkSettings resource
    ```bash
    az resource create \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $NETWORK_SETTINGS_RESOURCE_NAME \
      --resource-type GitHub.Network/networkSettings \
      --properties "{ \"location\": \"$AZURE_LOCATION\", \"properties\" : { \"subnetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP_NAME/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$SUBNET_NAME\", \"businessId\": \"$DATABASE_ID\" }}" \
      --is-full-object \
      --output table \
      --query "{GitHubId:tags.GitHubId, name:name}" \
      --api-version 2024-04-02
    ```

7. Copy the `GitHubId` value from the output — needed in Phase 2

---

## Phase 2 — GitHub Configuration

1. Go to **Enterprise Settings → Hosted compute networking → New network configuration → Azure private network**
2. Paste the `GitHubId` (NetworkSettings resource ID) from Phase 1 Step 7
3. Create a runner group → select the network configuration under **Network configurations**
4. Add a GitHub-hosted larger runner to the runner group
5. *(Optional)* Enable org-level network configuration: **Enterprise → Policies → Hosted compute networking → Enable** — allows org owners to create their own configurations

---

## Teardown Order (if needed)

> ⚠️ **Order matters** — skipping steps will leave a Service Association Link on the subnet blocking deletion.

1. Delete the runner from the runner group
2. Delete the runner group
3. Delete the network configuration in GitHub
4. Delete the NetworkSettings resource:
    ```bash
    az resource delete \
      -g $RESOURCE_GROUP_NAME \
      --name $NETWORK_SETTINGS_RESOURCE_NAME \
      --resource-type 'GitHub.Network/networkSettings' \
      --api-version 2024-04-02
    ```
5. Delete the subnet in Azure
