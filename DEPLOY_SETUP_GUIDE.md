# Deployment Setup Guide — Azure Foundry to SharePoint, Full System

End-to-end walkthrough for deploying this whole system from a clean Azure
subscription. Every step is a copy-pasteable command. Read this start-to-end
before you run anything; the order matters.

> **Audience:** developers running this by hand, terminal-first. Portal
> instructions are only used where there is no working CLI alternative
> (App Registration consent, the "Deploy to Azure" button as an optional
> alternative to `az deployment group create`).

---

## 0. What you're building

```
┌─────────────────────────── Hub VNet (10.0.0.0/16) ──────────────────────────┐
│                                                                              │
│    AzureFirewallSubnet ───── Azure Firewall ───── Public IP                  │
│                                  │                                           │
│    7 Private DNS zones           │  forced tunnel (UDR 0.0.0.0/0 → FW IP)    │
│    (privatelink.cognitiveservices, openai, services.ai, search.windows,     │
│     blob.core, file.core, documents.azure, vaultcore)                       │
│                                  │                                           │
└──────────────────── peering ─────┼─────────────────────────────────────────┘
                                   │
┌─────────────────────────── Spoke VNet (10.100.0.0/16) ─────────────────────┐
│                                                                              │
│  agent-subnet (delegated)   pe-subnet            vm-subnet     bastion       │
│   AI Foundry Agent Service   private endpoints    jumpbox        │           │
│                                  │                                           │
│  Foundry (private):              │                                           │
│   ├ AI Foundry account + project (cognitiveservices PE)                     │
│   ├ AI Foundry agents (services.ai PE)                                      │
│   ├ Azure OpenAI (openai PE)  ── gpt-4.1 + text-embedding-3-small           │
│   ├ Azure AI Search Standard  (search PE) ── private vector index            │
│   ├ Azure Storage             (blob/file PE) ── SharePoint mirror container  │
│   └ Azure Cosmos DB           (documents PE) ── thread state                 │
│                                                                              │
│  SharePoint sync layer (this repo creates):                                  │
│   ├ Function App (.NET 10 isolated, Flex Consumption, VNet-integrated)       │
│   ├ Function-storage (private, identity-based)                               │
│   ├ Key Vault (private, RBAC) ── stores SPN secret                           │
│   └ Function timers + sync_ui HTTP trigger                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                          Microsoft Graph / SharePoint Online
                          (egress through Azure Firewall)
```

The SharePoint sync function reads files + permissions from one or more
SharePoint sites via Microsoft Graph, mirrors them into the storage account
with permission metadata as ACL fields, and an Azure AI Search indexer chunks
+ embeds + indexes them with a dual-vector / semantic schema. A Foundry agent
queries that index via the `azure_ai_search` tool with security trimming on
`acl_user_ids` / `acl_group_ids`. Optionally a second agent uses agentic
retrieval (Knowledge Base + planner) over the same index.

---

## 1. Before you start — accounts, roles, quota

### 1.1 Azure subscription + role

You need an Azure subscription where you (the deployer) have:

- **Contributor** on the subscription, AND
- **User Access Administrator** on the subscription (the deploy script creates
  RBAC role assignments — `Storage Blob Data Contributor`,
  `Cognitive Services OpenAI User`, `Search Index Data Reader`, etc.)

Easier: **Owner** on the subscription. Easier still: **Owner on a dedicated
resource-group scope** if your org doesn't allow subscription-level Owner.

### 1.2 Microsoft Entra (Azure AD) tenant access

For the SharePoint sync app registration you need:

- The ability to **create app registrations** in your Entra tenant, AND
- **A Directory admin** who can grant **admin consent** for Microsoft Graph
  application permissions (you can be that person, or you'll hand the consent
  link to the admin once).

If you can't get admin consent, this whole project doesn't work — Microsoft
Graph application permissions (`Files.Read.All`, `Sites.Read.All`,
`InformationProtectionPolicy.Read.All`) all require it.

### 1.3 SharePoint Online site to sync

You need at least one SharePoint Online site with files in a document library.
Save its full URL — for example
`https://contoso.sharepoint.com/sites/MySite`. You'll set this in
`sharepoint-sync.env` as `SHAREPOINT_SITE_URL`.

### 1.4 Azure region and quota

Pick **one Azure region** for everything (hub, spoke, Foundry). The repo is
tested in `swedencentral`, `eastus`, `francecentral`. Whichever region you
pick must have:

- **Azure OpenAI** with `gpt-4.1` (any 4.x chat model works) and
  `text-embedding-3-small` capacity available. You'll burn ~30k TPM for the
  chat model and a small amount for embeddings.
- **Azure AI Search Standard (S1)** SKU available — semantic search ships free
  on S1+.
- **Flex Consumption Function App** support.
- **Azure Container Apps** support (the Foundry Agent Service uses ACA
  environments behind the scenes).

### 1.5 Resource providers

Run this once per subscription. Idempotent — safe to repeat.

```bash
SUBSCRIPTION_ID=<your-subscription-id>
az login
az account set --subscription "$SUBSCRIPTION_ID"

for ns in \
  Microsoft.Network \
  Microsoft.Storage \
  Microsoft.Web \
  Microsoft.App \
  Microsoft.KeyVault \
  Microsoft.Search \
  Microsoft.CognitiveServices \
  Microsoft.OperationalInsights \
  Microsoft.Insights \
  Microsoft.ContainerRegistry \
  Microsoft.DocumentDB \
  Microsoft.AlertsManagement; do
  az provider register --namespace "$ns"
done

# Wait for them all to flip to "Registered" (~1-3 min)
for ns in Microsoft.Network Microsoft.App Microsoft.CognitiveServices; do
  echo "$ns: $(az provider show --namespace $ns --query registrationState -o tsv)"
done
```

### 1.6 Deploy machine

Use the WSL2 Ubuntu 24.04 setup described in
[`PREREQUISITES.md`](PREREQUISITES.md) and install the tools listed in
[`requirements.txt`](requirements.txt):

```
dotnet-sdk==10.0.7
azure-cli==2.85
azure-functions-core-tools==4.9
python==3.14.4
jq==1.8.1
gh==2.91
```

Plus `git`, `curl`, `wget`, `zip`/`unzip` (bundled with WSL).

Verify the install:

```bash
dotnet --list-sdks            # 10.0.107 [/usr/lib/dotnet/sdk]
dotnet --list-runtimes        # Microsoft.NETCore.App 10.0.7
az --version                  # 2.85.x
func --version                # 4.9.x
python3 --version             # 3.14.4
jq --version                  # jq-1.8.1
gh --version                  # 2.91.x
```

> **Important:** *Steps 0 → 4 below* (App Registration, hub, spoke, Foundry
> Bicep) can run from anywhere with internet access.
> ***Step 5 (`3-deploy-sharepoint-sync.sh`) MUST run from inside the spoke
> VNet*** — it talks to private endpoints. We deploy a Bastion + jumpbox VM
> for that purpose in step 3 and switch to it before step 5.

---

## 2. Clone the repo and build the .NET project

```bash
mkdir -p ~/projects/repos
cd ~/projects/repos
git clone git@github.com:eli-tectika/Azure-Foundry-to-SharePoint---Full-System.git
cd Azure-Foundry-to-SharePoint---Full-System

# Build once — this both validates your toolchain and pre-fills the NuGet cache
cd deployment/sharepoint-sync-func
dotnet restore
dotnet build -c Release
dotnet list package --vulnerable --include-transitive   # expect: no vulnerable
dotnet list package --deprecated --include-transitive   # expect: no deprecated
cd ../..

# Make every shell script executable (bash safe — preserves +x in WSL)
chmod +x deployment/*.sh \
         deployment/sharepoint-sync-func/deploy/*.sh \
         bicep/*.sh
```

If `dotnet build` fails — stop. Fix the toolchain before going further.

---

## 3. Phase 1 — Microsoft Entra App Registration for SharePoint

The Function App authenticates to Microsoft Graph with a service principal
(SPN). You create the app registration, add Graph permissions, get admin
consent for the tenant, and generate a client secret. **Three values come out
of this step that you'll plug into env files later:** `tenant_id`, `client_id`,
and `client_secret`.

### 3.1 Create the app registration

Pick a name. A common pattern is `<project>-sharepoint-sync-spn`.

```bash
APP_DISPLAY_NAME="azure-foundry-sharepoint-sync-spn"

APP_JSON=$(az ad app create --display-name "$APP_DISPLAY_NAME")
APP_ID=$(echo "$APP_JSON" | jq -r '.appId')
OBJECT_ID=$(echo "$APP_JSON" | jq -r '.id')

# Create the SPN tied to this app
az ad sp create --id "$APP_ID" --output none

TENANT_ID=$(az account show --query tenantId -o tsv)

echo "================ SAVE THESE ================"
echo "AZURE_TENANT_ID = $TENANT_ID"
echo "AZURE_CLIENT_ID = $APP_ID"
echo "==========================================="
```

### 3.2 Add Microsoft Graph application permissions

Microsoft Graph's app ID is fixed (`00000003-0000-0000-c000-000000000000`).
The permission GUIDs below are also fixed Microsoft IDs — they don't change
per tenant.

```bash
GRAPH_APP_ID="00000003-0000-0000-c000-000000000000"

# Application-permission GUIDs (well-known, do not change):
SITES_READ_ALL="332a536c-c7ef-4017-ab91-336970924f0d"           # Sites.Read.All
FILES_READ_ALL="01d4889c-1287-42c6-ac1f-5d1e02578ef6"           # Files.Read.All
INFOPROTPOL_READ="19da66cb-0fb0-4390-b071-ebc76a349482"         # InformationProtectionPolicy.Read.All

az ad app permission add --id "$APP_ID" --api "$GRAPH_APP_ID" --api-permissions \
  "$SITES_READ_ALL=Role" \
  "$FILES_READ_ALL=Role" \
  "$INFOPROTPOL_READ=Role"
```

> **Choosing `Sites.Read.All` vs `Sites.Selected`**: the snippet above grants
> tenant-wide `Sites.Read.All`. If your org's security policy requires
> per-site scoping, swap `Sites.Read.All` for **`Sites.Selected`** (GUID:
> `883ea226-0bf2-4a8f-9f9d-92c9162a727d`). After admin consent you'll then
> have to run the per-site grant in §3.5.

### 3.3 Grant admin consent

Easiest path — CLI:

```bash
az ad app permission admin-consent --id "$APP_ID"
```

If your account isn't a directory admin, the call returns `Forbidden`. In
that case, send this URL to a directory admin — when they sign in and click
Accept, every required permission is consented at once:

```bash
echo "https://login.microsoftonline.com/$TENANT_ID/adminconsent?client_id=$APP_ID"
```

Verify consent landed:

```bash
az ad app permission list-grants --id "$APP_ID" --show-resource-name
# Should list Microsoft Graph with the three permissions above.
```

### 3.4 Generate a client secret

```bash
# Two-year expiration — pick what your secret rotation policy allows
SECRET_JSON=$(az ad app credential reset --id "$APP_ID" --append \
  --display-name "deploy-secret-$(date +%Y%m%d)" --years 2)

CLIENT_SECRET=$(echo "$SECRET_JSON" | jq -r '.password')

echo "================ SAVE THIS NOW (only shown once) ================"
echo "AZURE_CLIENT_SECRET = $CLIENT_SECRET"
echo "================================================================="
```

> **Critical:** `--password` value is shown only once. If you lose it, run
> `az ad app credential reset` again (which generates a new one). Note: every
> reset *appends* a credential, so `--append` keeps the old ones alive until
> they expire.

### 3.5 (Sites.Selected only) Grant the app to specific sites

Skip this entire sub-section if you used `Sites.Read.All` in §3.2.

```bash
SHAREPOINT_SITE_URL="https://contoso.sharepoint.com/sites/MySite"

# Resolve the Graph site ID from the URL
HOST=$(echo "$SHAREPOINT_SITE_URL" | sed -E 's#https?://([^/]+).*#\1#')
PATH_PART=$(echo "$SHAREPOINT_SITE_URL" | sed -E 's#https?://[^/]+##')
SITE_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/sites/${HOST}:${PATH_PART}" \
  --query id -o tsv)
echo "Site ID: $SITE_ID"

# Grant read on this site to the app
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/sites/${SITE_ID}/permissions" \
  --headers "Content-Type=application/json" \
  --body "{\"roles\":[\"read\"],\"grantedToIdentities\":[{\"application\":{\"id\":\"$APP_ID\",\"displayName\":\"$APP_DISPLAY_NAME\"}}]}"
```

Repeat the `az rest --method POST` block for every additional site you want
the function to be able to read.

---

## 4. Phase 2 — Deploy the hub network (`1-deploy-hub.sh`)

This creates the hub resource group, hub VNet, AzureFirewallSubnet, Azure
Firewall + policy, Log Analytics workspace, and the seven private DNS zones.

### 4.1 Fill in `hub.env`

```bash
cd deployment
cp hub.env.example hub.env
${EDITOR:-vi} hub.env
```

Required values:

| Var | Example | Notes |
|---|---|---|
| `SUBSCRIPTION_ID` | (your subscription GUID) | from `az account show` |
| `LOCATION` | `swedencentral` | |
| `HUB_RG` | `foundry-hub-rg` | resource group name |
| `HUB_VNET_NAME` | `hub-vnet` | |
| `HUB_VNET_PREFIX` | `10.0.0.0/16` | hub address space |
| `HUB_FW_SUBNET_PREFIX` | `10.0.1.0/26` | min /26 — required by Azure Firewall |
| `FW_NAME` | `hub-firewall` | |
| `FW_POLICY_NAME` | `hub-fw-policy` | spoke + sync scripts reference this |
| `FW_PIP_NAME` | `hub-fw-pip` | |
| `LAW_NAME` | `hub-fw-law` | Log Analytics workspace |

Optional:

| Var | Default | Notes |
|---|---|---|
| `FW_SKU` | `Standard` | `Basic` is cheaper but doesn't support TLS inspection |
| `LAW_RETENTION_DAYS` | `30` | |

### 4.2 Run it

```bash
./1-deploy-hub.sh
```

Takes ~10–15 minutes — Azure Firewall provisioning is the slow part. The
script ends with a summary block. **Copy down the firewall private IP** from
the output — you'll need it later:

```bash
FW_PRIVATE_IP=$(az network firewall show \
  -g "$HUB_RG" -n "$FW_NAME" \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)
echo "FW_PRIVATE_IP=$FW_PRIVATE_IP"
```

### 4.3 Verify

```bash
# All seven DNS zones exist in the hub RG
for zone in \
  privatelink.cognitiveservices.azure.com \
  privatelink.openai.azure.com \
  privatelink.services.ai.azure.com \
  privatelink.search.windows.net \
  privatelink.documents.azure.com \
  privatelink.blob.core.windows.net \
  privatelink.file.core.windows.net \
  privatelink.vaultcore.azure.net; do
  az network private-dns zone show -n "$zone" -g "$HUB_RG" \
      --query name -o tsv 2>/dev/null \
      && echo "✅ $zone" \
      || echo "❌ $zone MISSING — re-run 1-deploy-hub.sh"
done

# Firewall is running
az network firewall show -g "$HUB_RG" -n "$FW_NAME" \
  --query "{name:name, sku:sku.tier, state:provisioningState, ip:ipConfigurations[0].privateIPAddress}" -o table
```

---

## 5. Phase 3 — Deploy the spoke network (`2-deploy-spoke.sh`)

Creates the spoke resource group, spoke VNet with subnets, links the seven
DNS zones to the spoke VNet, sets up VNet peering both ways, and (by default)
deploys an Azure Bastion + an Ubuntu jumpbox VM. The jumpbox is what you'll
use to run step 7 (`3-deploy-sharepoint-sync.sh`).

### 5.1 Fill in `spoke.env`

```bash
cp spoke.env.example spoke.env
${EDITOR:-vi} spoke.env
```

`HUB_RG`, `HUB_VNET_NAME`, `LOCATION`, `SUBSCRIPTION_ID` must match
`hub.env` exactly.

| Var | Example | Notes |
|---|---|---|
| `SPOKE_RG` | `foundry-spoke-rg` | |
| `SPOKE_VNET_NAME` | `spoke-vnet` | |
| `SPOKE_VNET_PREFIX` | `10.100.0.0/16` | must not overlap hub |
| `SPOKE_AGENT_SUBNET_PREFIX` | `10.100.3.0/24` | delegated to `Microsoft.App/environments` |
| `SPOKE_PE_SUBNET_PREFIX` | `10.100.4.0/24` | private endpoints land here |
| `SPOKE_VM_SUBNET_PREFIX` | `10.100.2.0/24` | jumpbox subnet (optional but strongly recommended) |
| `SPOKE_BASTION_SUBNET_PREFIX` | `10.100.1.0/26` | Bastion subnet — min /26 |
| `DEPLOY_BASTION` | `true` | leave on so you get a jumpbox |
| `VM_ADMIN_USER` | `azureadmin` | jumpbox login user |

### 5.2 Run it

```bash
./2-deploy-spoke.sh
```

Takes ~15–20 minutes (Bastion is the slow part).

When it prompts you for the jumpbox VM password, pick a strong one and save
it to your password manager — you'll log in via Bastion shortly.

### 5.3 Verify

```bash
# Peering established both directions
az network vnet peering list -g "$HUB_RG" --vnet-name "$HUB_VNET_NAME" -o table
az network vnet peering list -g "$SPOKE_RG" --vnet-name "$SPOKE_VNET_NAME" -o table

# UDR routes to firewall
az network route-table list -g "$SPOKE_RG" \
  --query "[].{name:name, routes:routes[?addressPrefix=='0.0.0.0/0'].nextHopIpAddress}" -o table

# Bastion is up
az network bastion list -g "$SPOKE_RG" -o table

# Jumpbox VM
VM_NAME=$(az vm list -g "$SPOKE_RG" --query "[?contains(name,'test-vm')||contains(name,'jump')].name | [0]" -o tsv)
echo "Jumpbox VM: $VM_NAME"
```

---

## 6. Phase 4 — Deploy Foundry resources via Bicep

The hub + spoke scripts only created network plumbing. The actual Foundry
account, AI Search, Storage, Cosmos DB, and the gpt-4.1 model deployment come
from `bicep/main.bicep` (Microsoft's "Template 15 — private network standard
agent setup", with local UDR/firewall additions).

### 6.1 Edit `bicep/main.bicepparam`

Open `bicep/main.bicepparam` and set the parameters that reference your spoke
network:

```bash
${EDITOR:-vi} ../bicep/main.bicepparam
```

Set these explicitly to match your deployment:

| Param | Set to |
|---|---|
| `location` | same region as hub/spoke (e.g. `swedencentral`) |
| `existingVnetResourceId` | `/subscriptions/<sub>/resourceGroups/<SPOKE_RG>/providers/Microsoft.Network/virtualNetworks/<SPOKE_VNET_NAME>` |
| `agentSubnetName` | `agent-subnet` (matches what `2-deploy-spoke.sh` created) |
| `peSubnetName` | `pe-subnet` |
| `firewallPrivateIp` | the value you saved at the end of §4.2 (e.g. `10.0.1.4`) |
| `dnsZonesSubscriptionId` | same as your `SUBSCRIPTION_ID` (or hub-DNS subscription if you have a central-DNS topology) |
| `existingDnsZones` | for each zone, set the value to the resource ID of the zone in the hub RG (see snippet below) |
| `firstProjectName` | `project` (or whatever you want) |
| `modelCapacity` | `30` (or higher if quota allows) |

The `existingDnsZones` block needs full resource IDs. Generate them:

```bash
# Run from your repo root
HUB_RG=foundry-hub-rg     # match hub.env
SUB=$SUBSCRIPTION_ID

for zone in \
  privatelink.services.ai.azure.com \
  privatelink.openai.azure.com \
  privatelink.cognitiveservices.azure.com \
  privatelink.search.windows.net \
  privatelink.blob.core.windows.net \
  privatelink.documents.azure.com; do
  ID=$(az network private-dns zone show -n "$zone" -g "$HUB_RG" --query id -o tsv 2>/dev/null)
  echo "  '$zone': '$ID'"
done
```

Paste each line into the `existingDnsZones` block in
`main.bicepparam`. Example:

```bicep
param existingDnsZones = {
  'privatelink.services.ai.azure.com': '/subscriptions/.../privateDnsZones/privatelink.services.ai.azure.com'
  'privatelink.openai.azure.com': '/subscriptions/.../privateDnsZones/privatelink.openai.azure.com'
  // ...etc
}
```

### 6.2 Deploy the Bicep

```bash
SPOKE_RG=foundry-spoke-rg

az deployment group create \
  --resource-group "$SPOKE_RG" \
  --template-file ../bicep/main.bicep \
  --parameters ../bicep/main.bicepparam
```

Takes ~20–30 minutes. The deployment creates:

- **Foundry account** (Cognitive Services kind=AIServices) with a project
- **gpt-4.1** model deployment (capacity = `modelCapacity`)
- **text-embedding-3-small** model deployment
- **Azure AI Search** Standard SKU
- **Storage account** (StorageV2) for the SharePoint mirror
- **Azure Cosmos DB** for thread/conversation state
- **Azure Key Vault**
- Private endpoints + DNS A-records for every one of the above
- Project capability host (`caphostproj`) on the agent subnet

### 6.3 Save the resource names you'll need

```bash
SPOKE_RG=foundry-spoke-rg

AI_SERVICES_NAME=$(az cognitiveservices account list -g "$SPOKE_RG" \
  --query "[?kind=='AIServices'] | [0].name" -o tsv)

SEARCH_SERVICE_NAME=$(az search service list -g "$SPOKE_RG" \
  --query "[0].name" -o tsv)

STORAGE_ACCOUNT_NAME=$(az storage account list -g "$SPOKE_RG" \
  --query "[?starts_with(name,'st') && !contains(name,'func')] | [0].name" -o tsv)

FOUNDRY_PROJECT_ENDPOINT=$(az cognitiveservices account show \
  -g "$SPOKE_RG" -n "$AI_SERVICES_NAME" \
  --query "properties.endpoints.\"AI Foundry API\"" -o tsv)

echo "================ SAVE THESE ================"
echo "AI_SERVICES_NAME=$AI_SERVICES_NAME"
echo "SEARCH_SERVICE_NAME=$SEARCH_SERVICE_NAME"
echo "STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME"
echo "FOUNDRY_PROJECT_NAME=project"
echo "FOUNDRY_PROJECT_ENDPOINT=$FOUNDRY_PROJECT_ENDPOINT"
echo "OPENAI_RESOURCE_URI=https://$AI_SERVICES_NAME.cognitiveservices.azure.com/"
echo "==========================================="
```

### 6.4 (Manual fallback) If Bicep deploy fails on the capability host

The `createCapHost.sh` and `deleteCapHost.sh` scripts in `bicep/` are
fallbacks if the cap-host gets stuck. Normally you don't need them.

If the Bicep ends with an error like `capability host failed`:

```bash
cd ../bicep
./createCapHost.sh
# Answer prompts:
#   Subscription ID: <your sub>
#   Resource Group:  <SPOKE_RG>
#   Foundry Account or Project: <AI_SERVICES_NAME>
#   CapabilityHost name: caphostproj
#   Customer subnet ResourceId: /subscriptions/.../virtualNetworks/<SPOKE_VNET_NAME>/subnets/agent-subnet
```

If it gets *really* stuck and won't delete:

```bash
./deleteCapHost.sh
# Same prompts as above
```

---

## 7. Phase 5 — Set up the jumpbox so the next script can reach private endpoints

`3-deploy-sharepoint-sync.sh` calls AI Search and Storage data-plane APIs
directly. Both have public access disabled. So **this script must run from a
machine inside the spoke VNet**. The Bastion + Ubuntu VM created in §5.2 is
that machine.

### 7.1 Connect to the jumpbox via Bastion

```bash
SPOKE_RG=foundry-spoke-rg
VM_NAME=$(az vm list -g "$SPOKE_RG" --query "[0].name" -o tsv)
BASTION_NAME=$(az network bastion list -g "$SPOKE_RG" --query "[0].name" -o tsv)

az network bastion ssh \
  --name "$BASTION_NAME" \
  --resource-group "$SPOKE_RG" \
  --target-resource-id "$(az vm show -g $SPOKE_RG -n $VM_NAME --query id -o tsv)" \
  --auth-type password \
  --username azureadmin
```

(Enter the VM admin password you set during step 5.2.)

You should now have a shell inside the jumpbox VM.

### 7.2 Install the deploy toolchain on the jumpbox

The jumpbox is fresh Ubuntu 24.04. Install the same tools listed in
[`requirements.txt`](requirements.txt):

```bash
# === Run inside the jumpbox shell ===

sudo apt-get update
sudo apt-get install -y \
  git curl wget unzip zip jq \
  ca-certificates gnupg lsb-release apt-transport-https \
  python3 python3-pip

# .NET SDK 10 (Ubuntu 24.04 native repo — no MS repo needed)
sudo apt-get install -y dotnet-sdk-10.0

# Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az extension add --name containerapp --upgrade
az config set extension.use_dynamic_install=yes_without_prompt

# Azure Functions Core Tools v4
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | \
  sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg >/dev/null
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-$(lsb_release -cs)-prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-get update
sudo apt-get install -y azure-functions-core-tools-4

# GitHub CLI
sudo apt-get install -y gh

# Verify
dotnet --list-sdks    # 10.0.107
az --version          # 2.85+
func --version        # 4.9+
jq --version          # jq-1.8.1
```

### 7.3 Clone the repo onto the jumpbox

You'll need the same code on the jumpbox. Two options — pick one:

**Option A: clone with HTTPS + GitHub CLI** (simpler):

```bash
gh auth login        # follow prompts; pick HTTPS
git clone https://github.com/eli-tectika/Azure-Foundry-to-SharePoint---Full-System.git
cd Azure-Foundry-to-SharePoint---Full-System
chmod +x deployment/*.sh deployment/sharepoint-sync-func/deploy/*.sh bicep/*.sh
```

**Option B: copy from your local machine via Bastion tunnel** (skip if Option
A worked).

### 7.4 Authenticate Azure CLI on the jumpbox

```bash
az login --use-device-code      # opens a code you paste at microsoft.com/devicelogin
az account set --subscription "<your-subscription-id>"
```

### 7.5 Verify connectivity to the private endpoints

```bash
# These should resolve to RFC1918 (10.x / 172.16-31.x / 192.168.x) addresses,
# NOT public IPs. If they resolve to public IPs, your DNS zone links are
# wrong — go back to Phase 2/3 and fix.

STORAGE_ACCOUNT_NAME=<from §6.3>
SEARCH_SERVICE_NAME=<from §6.3>
AI_SERVICES_NAME=<from §6.3>

dig +short ${STORAGE_ACCOUNT_NAME}.blob.core.windows.net
dig +short ${SEARCH_SERVICE_NAME}.search.windows.net
dig +short ${AI_SERVICES_NAME}.cognitiveservices.azure.com
```

---

## 8. Phase 6 — Deploy the SharePoint sync pipeline (`3-deploy-sharepoint-sync.sh`)

This is the big one. From the jumpbox:

```bash
cd ~/Azure-Foundry-to-SharePoint---Full-System/deployment
cp sharepoint-sync.env.example sharepoint-sync.env
${EDITOR:-vi} sharepoint-sync.env
```

### 8.1 Required values in `sharepoint-sync.env`

Plug in the values you saved in earlier sections:

| Var | From |
|---|---|
| `SUBSCRIPTION_ID` | §1.5 |
| `LOCATION` | matches hub/spoke |
| `SPOKE_RG` | §5 |
| `SPOKE_VNET_NAME` | §5 |
| `HUB_RG` | §4 |
| `FW_PRIVATE_IP` | §4.2 |
| `SPOKE_ADDRESS_SPACE` | the spoke CIDR (e.g. `10.100.0.0/16`) |
| `AZURE_TENANT_ID` | §3.1 |
| `AZURE_CLIENT_ID` | §3.1 |
| `AZURE_CLIENT_SECRET` | §3.4 |
| `SHAREPOINT_SITE_URL` | your SharePoint site |
| `SHAREPOINT_DRIVE_NAME` | usually `Documents` or `Shared Documents` |
| `SHAREPOINT_FOLDER_PATH` | `/` for the whole drive, or `/HR,/ENG` for specific folders |
| `AZURE_STORAGE_ACCOUNT_NAME` | §6.3 |
| `AZURE_BLOB_CONTAINER_NAME` | `sharepoint-sync` (created if missing) |
| `SEARCH_SERVICE_NAME` | §6.3 |
| `SEARCH_RESOURCE_GROUP` | same as `SPOKE_RG` |
| `OPENAI_RESOURCE_URI` | §6.3 (`https://<AI_SERVICES_NAME>.cognitiveservices.azure.com/`) |
| `EMBEDDING_DEPLOYMENT_ID` | `text-embedding-3-small` |
| `EMBEDDING_MODEL_NAME` | `text-embedding-3-small` |
| `EMBEDDING_DIMENSIONS` | `1536` |
| `FOUNDRY_PROJECT_NAME` | `project` (matches the Bicep `firstProjectName`) |

Optional but commonly tuned:

| Var | Default | When to change |
|---|---|---|
| `SYNC_PERMISSIONS` | `true` | only flip off if your scenario doesn't need ACL trimming |
| `SYNC_PURVIEW_PROTECTION` | `false` | flip on if you have Purview labels and added `InformationProtectionPolicy.Read.All` |
| `DELETE_ORPHANED_BLOBS` | `true` | leave on |
| `SOFT_DELETE_ORPHANED_BLOBS` | `true` | leave on — safer than hard delete |
| `DRY_RUN` | `false` | flip to `true` for the first run of a new site to preview without touching blobs |
| `TIMER_SCHEDULE` | `0 0 * * * *` | hourly — change to `0 */15 * * * *` for every-15-min, etc. |
| `TIMER_SCHEDULE_FULL` | `0 0 3 * * *` | daily 03:00 UTC reconcile — change to weekly if your library is huge |
| `INDEXER_SCHEDULE_INTERVAL` | `PT1H` | matches `TIMER_SCHEDULE`. Use `PT15M`, `P1D`, etc. (ISO-8601) |
| `AGENT_QUERY_TYPE` | `semantic` | other choices: `simple`, `vector`, `vectorSemanticHybrid` |
| `USE_AGENTIC_RETRIEVAL` | `false` | set `true` to also create a Knowledge Base + agentic agent (Phase 8) |

### 8.2 Run the deploy

```bash
./3-deploy-sharepoint-sync.sh
```

Takes ~25–40 minutes. The script is idempotent — if anything fails part-way,
fix the input and re-run; finished steps are skipped or updated in place.

What it does, in order (each shows a banner in the output):

| Step | What it does |
|---|---|
| 0 | Auto-discovers UDR, PE subnet, DNS zone resource groups, firewall policy. Validates everything exists. |
| 1 | Creates `func-subnet` (delegated to `Microsoft.App/environments`) in the spoke VNet. |
| 2 | Creates the `sharepoint-sync` blob container in the Foundry storage account. |
| 3 | Creates a separate **function-storage** account (private, identity-based). |
| 4 | Adds private endpoints (blob, file, queue, table) for the function-storage. Waits for DNS propagation. |
| 5 | Creates the Function App (Flex Consumption, .NET 10 isolated, system MI, VNet-integrated). Retries up to 5 times. |
| 6 | Creates a private Key Vault and stores the SPN secret in it. |
| 7 | Configures Function App settings (env vars wired through to the worker). |
| 8 | Grants the Function MI `Storage Blob Data Contributor` on the Foundry storage. |
| 9 | Creates **Shared Private Links** — AI Search → Foundry Storage, AI Search → AI Services. Auto-approves them. |
| 10 | Grants AI Search MI `Cognitive Services OpenAI User` on AI Services. |
| 11 | Creates the AI Search **index** (vector + semantic), **data source** (with soft-delete column), **skillset** (OCR → merge → split → embed), and **indexer** (private execution, hourly). |
| 11b | Patches every indexer in the service to private-execution mode. |
| 12 | Adds Azure Firewall rules for Microsoft Graph + SharePoint + Entra ID egress. |
| 13 | `dotnet publish` the .NET worker, zip it, deploy via SCM `/api/publish`. Retries up to 5 times. |
| 14 | Creates the Foundry agent (`sharepoint-search-agent`) wired to the AI Search index via `azure_ai_search` tool. |
| 14b | (only if `USE_AGENTIC_RETRIEVAL=true`) Creates a Knowledge Source, Knowledge Base, project connection, and a second agent (`sharepoint-agentic`). |

At the end the script appends two values to `sharepoint-sync.env`:

```
SYNC_CONSOLE_URL=https://func-spsync-xxxxx.azurewebsites.net/api/sync?code=...
FUNC_APP_HOSTNAME=func-spsync-xxxxx.azurewebsites.net
```

Save these. `SYNC_CONSOLE_URL` is your manual sync trigger.

### 8.3 If a step fails

The script's output tells you which step failed. Common cases:

| Symptom | Fix |
|---|---|
| Step 0: "Hub RG not found" / "DNS zone not found" | re-check `HUB_RG`, `DNS_SUBSCRIPTION` in `sharepoint-sync.env` against your hub deployment |
| Step 5: "Function App create failed" after 5 retries | usually transient; wait 5 min, re-run |
| Step 9: "SPL approval timed out" | run the script again — SPL provisioning sometimes takes >60s and the second run picks up the now-approved SPL |
| Step 11: "Toggle public access denied (403)" | your subscription's Azure Policy is blocking public-access toggles. Set `SEARCH_ACCESS_MODE=private` in `sharepoint-sync.env`; the script will assume you're already inside the VNet (which from the jumpbox you are) |
| Step 13: "Publish failed: 'not authorized'" | RBAC propagation lag. Wait 5 min and re-run |
| Step 14: "Agent create failed" | confirm `FOUNDRY_PROJECT_NAME` is exactly `project` (or whatever you set in Bicep), and that the Bicep deploy fully completed |

---

## 9. Phase 7 — Verify end-to-end

### 9.1 Trigger a manual sync

From your laptop or the jumpbox — `SYNC_CONSOLE_URL` is publicly reachable
(it's auth'd by function master key; the function itself runs inside the VNet
and is what reaches private endpoints):

```bash
# Open in a browser:
echo "$SYNC_CONSOLE_URL"

# Or POST from the CLI (delta sync):
curl -X POST "${SYNC_CONSOLE_URL}&mode=delta" | jq
# → JSON with {ok: true, files_added: N, ...}

# Or full reconcile:
curl -X POST "${SYNC_CONSOLE_URL}&mode=full" | jq
```

The first run is **always a full enumeration** (the function has no delta
token yet), so it'll be slow — minutes, not seconds. Subsequent delta runs
should finish in seconds for small libraries.

### 9.2 Watch the function logs

```bash
FUNCTION_APP_NAME=$(echo "$FUNC_APP_HOSTNAME" | cut -d. -f1)

az functionapp log tail \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$SPOKE_RG"
# Ctrl-C to stop tailing
```

Look for `Sync completed (...)` and the file counts from the SyncStats.

### 9.3 Confirm blobs landed

```bash
az storage blob list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --container-name sharepoint-sync \
  --auth-mode login \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table | head -20
```

### 9.4 Confirm the AI Search indexer ran

```bash
SEARCH_KEY=$(az search admin-key show \
  --service-name "$SEARCH_SERVICE_NAME" -g "$SPOKE_RG" \
  --query primaryKey -o tsv)

curl -s "https://${SEARCH_SERVICE_NAME}.search.windows.net/indexers/sharepoint-blob-indexer/status?api-version=2025-11-01" \
  -H "api-key: $SEARCH_KEY" \
  | jq '.lastResult | {status, itemsProcessed, itemsFailed, errorMessage}'
```

`status: "success"` and `itemsProcessed > 0` means the indexer finished a run
and chunked + embedded your documents.

### 9.5 Test the Foundry agent — option A: agent-tool console app

From your local machine (not the jumpbox — the `agent-tool` queries Foundry
via the public API gateway, which works from anywhere):

```bash
cd ~/projects/repos/Azure-Foundry-to-SharePoint---Full-System/agent-tool
dotnet run -- \
  --endpoint "$FOUNDRY_PROJECT_ENDPOINT" \
  --test "List the documents available in the knowledge base"
```

Expect a textual response with citations to SharePoint URLs.

### 9.6 Test the Foundry agent — option B: Foundry portal Playground

1. Open https://ai.azure.com → your project
2. Sidebar → **Agents** → click `sharepoint-search-agent`
3. Click **Try in Playground**
4. Ask: "What documents do we have about <topic>?"

You should get answers grounded in your SharePoint content with clickable
citations to the actual SharePoint URLs.

---

## 10. (Optional) Phase 8 — Agentic retrieval prototype

Skip this section if you're not ready for the agentic path.

If you want a Knowledge Base + planner agent (creates a second agent that
coexists with the primary one):

```bash
# On the jumpbox (it needs VNet access to the AI Search private endpoint):
cd ~/Azure-Foundry-to-SharePoint---Full-System/deployment

# Either set USE_AGENTIC_RETRIEVAL=true in sharepoint-sync.env and re-run the
# main deploy script, OR run just the prototype:
./test-agentic-retrieval.sh

# Cleanup later if needed:
CLEANUP=1 ./test-agentic-retrieval.sh
```

This creates `sharepoint-ks` (Knowledge Source), `sharepoint-kb` (Knowledge
Base), a project connection, and a `sharepoint-agentic` agent. Test it the
same ways as §9.5/9.6.

---

## 11. Day-2 operations

### 11.1 Manual sync triggers

```bash
# Delta (fast incremental)
curl -X POST "${SYNC_CONSOLE_URL}&mode=delta"

# Full reconcile (catches renames + orphans)
curl -X POST "${SYNC_CONSOLE_URL}&mode=full"
```

### 11.2 Force a fresh full crawl

The function persists a Graph delta token in
`sharepoint-sync/.sync-state/delta-token.json`. To force a full re-crawl
(e.g. after rotating the SPN secret or testing):

```bash
az storage blob delete \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --container-name sharepoint-sync \
  --name ".sync-state/delta-token.json" \
  --auth-mode login
```

The next run will be a full delta crawl, then resume incrementally.

### 11.3 Adjust the schedule

```bash
# Every 15 minutes for delta, daily 04:00 UTC for full reconcile:
az functionapp config appsettings set \
  --name "$FUNCTION_APP_NAME" -g "$SPOKE_RG" \
  --settings \
    "TIMER_SCHEDULE=0 */15 * * * *" \
    "TIMER_SCHEDULE_FULL=0 0 4 * * *"
```

### 11.4 Read logs

```bash
# Function App: live tail
az functionapp log tail --name "$FUNCTION_APP_NAME" -g "$SPOKE_RG"

# AI Search indexer status
curl -s "https://${SEARCH_SERVICE_NAME}.search.windows.net/indexers/sharepoint-blob-indexer/status?api-version=2025-11-01" \
  -H "api-key: $SEARCH_KEY" | jq

# Application Insights (if APPLICATIONINSIGHTS_CONNECTION_STRING is set):
APP_INSIGHTS_NAME=<from your portal>
az monitor app-insights query \
  --app "$APP_INSIGHTS_NAME" -g "$SPOKE_RG" \
  --analytics-query "traces | where timestamp > ago(1h) | order by timestamp desc | take 100"
```

### 11.5 Rotate the SPN secret

When the secret you generated in §3.4 nears expiration:

```bash
NEW_SECRET=$(az ad app credential reset --id "$AZURE_CLIENT_ID" --append \
  --display-name "rotate-$(date +%Y%m%d)" --years 2 --query password -o tsv)

# Update Key Vault
az keyvault secret set \
  --vault-name "$KV_NAME" \
  --name "sp-client-secret" \
  --value "$NEW_SECRET" --output none

# Restart the Function App so it picks up the new value
az functionapp restart --name "$FUNCTION_APP_NAME" -g "$SPOKE_RG"
```

The old secret keeps working until it expires (no downtime).

---

## 12. Troubleshooting

### 12.1 The function logs an OAuth/Graph 401

- SPN secret wrong or expired → §11.5
- Admin consent missing → §3.3
- App Registration not in same tenant as your `AZURE_TENANT_ID` env value

### 12.2 The function logs `itemNotFound` or `403` on a SharePoint folder

- For `Sites.Selected` — that specific site isn't granted to this app yet → §3.5
- For `Sites.Read.All` — the site has been deleted, renamed, or is in a
  tenant your account can't see. The function will skip and continue per
  recent fixes; check the warning in the logs

### 12.3 The Function App can't reach `*.sharepoint.com`

- Step 12 of the deploy script adds the Graph + SharePoint + Entra ID
  firewall rules. Re-run the script if it failed mid-way
- For 3rd-party firewall environments (`FW_MODE=external` in `sharepoint-sync.env`),
  the script prints the FQDN list — add them to your NVA manually

### 12.4 AI Search indexer status is `transientFailure`

- Almost always: indexer's `executionEnvironment` reverted to `standard`.
  Step 11b of the deploy script patches every indexer to `private`. Re-run
  the deploy or run this directly:

```bash
curl -X PUT "https://${SEARCH_SERVICE_NAME}.search.windows.net/indexers/sharepoint-blob-indexer?api-version=2025-11-01" \
  -H "api-key: $SEARCH_KEY" -H "Content-Type: application/json" \
  --data "$(curl -s "https://${SEARCH_SERVICE_NAME}.search.windows.net/indexers/sharepoint-blob-indexer?api-version=2025-11-01" -H "api-key: $SEARCH_KEY" | jq '.parameters.configuration.executionEnvironment="private" | del(.\"@odata.etag\")')"
```

### 12.5 Foundry agent returns `doc_0` placeholders instead of real URLs

The default `FOUNDRY_AGENT_INSTRUCTIONS` enforces bare-URL citations. If
you've overridden them, make sure your custom prompt still asks the model to
emit raw `https://contoso.sharepoint.com/...` URLs (not numbered placeholders
like `[1]`). Re-run the deploy script after editing the env file.

### 12.6 Bicep deploy fails on the project capability host

→ §6.4 — run `bicep/createCapHost.sh` interactively, or `deleteCapHost.sh` to
clear a stuck one and let Bicep recreate it.

### 12.7 RBAC role assignment fails with `AuthorizationFailed`

You need **User Access Administrator** on the subscription (in addition to
Contributor) to create role assignments. See §1.1.

### 12.8 The deploy script keeps running but never finishes Step 9 (SPL)

Approve manually:

```bash
SEARCH_ID=$(az search service show -g "$SPOKE_RG" -n "$SEARCH_SERVICE_NAME" --query id -o tsv)
STORAGE_ID=$(az storage account show -g "$SPOKE_RG" -n "$STORAGE_ACCOUNT_NAME" --query id -o tsv)

PE_CONN_ID=$(az network private-endpoint-connection list \
  --id "$STORAGE_ID" \
  --query "[?contains(name,'search')] | [0].id" -o tsv)

az network private-endpoint-connection approve \
  --id "$PE_CONN_ID" --description "approved for SharePoint sync"
```

Then re-run the deploy script.

---

## 13. Tearing it all down

There's no automated teardown. Manual cleanup:

```bash
# Just the SharePoint sync (keep Foundry + network):
az functionapp delete -n "$FUNCTION_APP_NAME" -g "$SPOKE_RG"
az keyvault delete -n "$KV_NAME" -g "$SPOKE_RG"
# (optional) az keyvault purge -n "$KV_NAME" --location "$LOCATION"
az storage account delete -n "$FUNC_STORAGE_NAME" -g "$SPOKE_RG" --yes

# Everything (nukes spoke and hub):
az group delete -n "$SPOKE_RG" --yes --no-wait
az group delete -n "$HUB_RG" --yes --no-wait

# The App Registration is at tenant level, not in any RG:
az ad app delete --id "$AZURE_CLIENT_ID"
```

Soft-deleted Key Vaults stay around for 90 days — purge them if you'll redeploy
with the same name immediately.

---

## 14. Reference

- Architecture / network design: [`README.md`](README.md) — long-form guide
- SharePoint sync function specifics:
  [`deployment/sharepoint-sync-func/README.md`](deployment/sharepoint-sync-func/README.md)
- Hub-spoke + Foundry deep dive:
  [`azure-ai-foundry-blog/posts/deploying-private-foundry-agent-hub-spoke.md`](azure-ai-foundry-blog/posts/deploying-private-foundry-agent-hub-spoke.md)
- SharePoint citations in Foundry agents:
  [`azure-ai-foundry-blog/posts/foundry-agent-sharepoint-citations.md`](azure-ai-foundry-blog/posts/foundry-agent-sharepoint-citations.md)
- Microsoft's original Bicep template (the basis of `bicep/main.bicep`):
  https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup

---

## 15. Phase summary (cheat sheet for re-deploys)

| Phase | Script / action | Where | Time |
|---|---|---|---|
| 0 | Install tools per [`PREREQUISITES.md`](PREREQUISITES.md) | local | 30 min |
| 1 | App Registration + Graph permissions + admin consent + secret | local | 10 min |
| 2 | `1-deploy-hub.sh` | local | 15 min |
| 3 | `2-deploy-spoke.sh` | local | 20 min |
| 4 | `az deployment group create ... main.bicep` | local | 30 min |
| 5 | Bastion + jumpbox toolchain install | jumpbox | 15 min |
| 6 | `3-deploy-sharepoint-sync.sh` | jumpbox | 35 min |
| 7 | Verify (manual sync, agent test) | local + jumpbox | 10 min |
| 8 | (optional) `test-agentic-retrieval.sh` | jumpbox | 10 min |

End-to-end first deploy: **~3 hours**. Subsequent re-deploys (idempotent
re-run of just `3-deploy-sharepoint-sync.sh`): ~10 minutes.
