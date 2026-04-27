# Deployment-Machine Prerequisites (Windows)

One-time setup for the Windows machine that will deploy this repository to
Azure. Install everything in this list in a single session, then verify with
the **Verification** section at the bottom.

The repository ships:
- A .NET 10 Azure Functions isolated-worker project (`deployment/sharepoint-sync-func/`)
- A .NET 10 console tool (`agent-tool/`) that creates / tests Foundry agents
- Bicep templates for hub-spoke + Foundry resources (`bicep/`)
- Bash deploy scripts (hub, spoke, sync) plus per-app deploy scripts under
  `deployment/sharepoint-sync-func/deploy/`

The deploy scripts are **bash**, so on Windows you need either **WSL2** (Option A,
recommended) or **Git Bash** (Option B, also works). Pick one; do not mix.

---

## A. Recommended path: WSL2 + Ubuntu

This is the cleanest setup — every script runs natively, no quoting/path
weirdness, and `az`/`dotnet`/`func` all behave as on the Linux build agents
Microsoft tests against.

### A.1 — Enable WSL2 and install Ubuntu

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot when prompted. On first launch, set a Linux username + password.

If WSL is already installed, just add the distro:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default Ubuntu-24.04
```

Verify:
```powershell
wsl --list --verbose      # should show Ubuntu-24.04, VERSION 2
wsl -d Ubuntu-24.04 -- uname -a
```

### A.2 — Install everything inside the Ubuntu shell

Open the Ubuntu app (or `wsl` from PowerShell), then run the commands below
**inside Ubuntu**. Run them all once; they are idempotent.

```bash
sudo apt-get update && sudo apt-get -y upgrade

# Core CLI / shell tools the deploy scripts call directly
sudo apt-get install -y \
  curl wget unzip zip jq git ca-certificates gnupg lsb-release \
  python3 python3-pip build-essential apt-transport-https
```

#### .NET SDK 10

Both projects (`SharePointSyncFunc.csproj` and `agent-tool/AgentTool.csproj`)
target `net10.0` (LTS). One SDK covers both.

```bash
# Microsoft package signing key + repo
wget https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb -O /tmp/pmp.deb
sudo dpkg -i /tmp/pmp.deb
sudo apt-get update

sudo apt-get install -y dotnet-sdk-10.0
```

#### Azure CLI

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Required Azure CLI extensions (auto-prompted on first use, but pre-install to
avoid interactive prompts in the deploy scripts):

```bash
az extension add --name containerapp --upgrade
az extension add --name application-insights --upgrade
az config set extension.use_dynamic_install=yes_without_prompt
```

#### Azure Functions Core Tools (`func` CLI)

The deploy scripts call `func azure functionapp publish ... --dotnet-isolated`.

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | \
  sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg >/dev/null
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-$(lsb_release -cs)-prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-get update
sudo apt-get install -y azure-functions-core-tools-4
```

#### Docker (only if you also build the ACA Job container image)

The Function-App path does NOT need Docker. Only install if you plan to use
`TARGET=aca` or `TARGET=both` in `deploy-new.sh` / `deploy-existing.sh`, or
the optional ACR image build inside `3-deploy-sharepoint-sync.sh`.

Two options:
1. **Docker Desktop on Windows** with WSL2 integration enabled (recommended;
   install on Windows, then Docker Desktop > Settings > Resources > WSL
   integration > toggle Ubuntu-24.04). Then `docker` is available inside
   Ubuntu without further setup.
2. **Docker inside Ubuntu directly** (no Docker Desktop). Follow the official
   guide: https://docs.docker.com/engine/install/ubuntu/

If you're not sure: pick option 1.

#### GitHub CLI (optional — only if you use `gh` for PRs / issues)

```bash
sudo apt-get install -y gh
gh auth login
```

#### Git config + SSH key for GitHub

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# SSH key for github.com
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub        # copy to https://github.com/settings/keys
ssh -T git@github.com            # confirm "Hi <user>!"
```

---

## B. Alternative path: Native Windows + Git Bash

Use this only if you cannot enable WSL2 (e.g. corporate policy). Everything
below installs as **Windows native**. Run the deploy scripts from **Git Bash**,
not from PowerShell or `cmd.exe`.

> Tip: install everything with `winget` from an elevated PowerShell so that
> versions are auditable and updates are scriptable.

### B.1 — Install via winget (run as Admin in PowerShell)

```powershell
# Shells / VCS / general tools
winget install --id Git.Git -e
winget install --id GitHub.cli -e
winget install --id Microsoft.VisualStudioCode -e

# .NET SDK 10 (both projects target net10.0)
winget install --id Microsoft.DotNet.SDK.10 -e

# Azure CLI
winget install --id Microsoft.AzureCLI -e

# Azure Functions Core Tools (provides the `func` command)
winget install --id Microsoft.AzureFunctionsCoreTools -e

# Python 3 (JSON-parsing helpers in the bash scripts call python3)
winget install --id Python.Python.3.12 -e

# jq (used by bicep/createCapHost.sh + deleteCapHost.sh)
winget install --id stedolan.jq -e

# 7zip (for any zip/unzip needs not covered by Git Bash)
winget install --id 7zip.7zip -e

# (optional) Docker Desktop — only needed for the ACA Job container path
winget install --id Docker.DockerDesktop -e
```

After installation, **close and reopen the terminal** so PATH updates pick up.

### B.2 — Required Azure CLI extensions

In PowerShell or Git Bash:

```bash
az extension add --name containerapp --upgrade
az extension add --name application-insights --upgrade
az config set extension.use_dynamic_install=yes_without_prompt
```

### B.3 — Make sure `python3` is callable from Git Bash

The bash scripts call `python3 -c "..."` for tiny JSON parses. Windows installs
Python as `python.exe`, not `python3.exe`. Fix this once:

```bash
# In Git Bash:
echo 'alias python3=python' >> ~/.bashrc
source ~/.bashrc
```

Or, in PowerShell (one-off symlink):
```powershell
New-Item -ItemType SymbolicLink -Path "$env:LOCALAPPDATA\Programs\Python\Python312\python3.exe" -Target "$env:LOCALAPPDATA\Programs\Python\Python312\python.exe"
```

### B.4 — Git config + SSH key for GitHub

Same as Option A's last step; use Git Bash.

---

## C. Azure-side prerequisites (both options)

The scripts assume these are already in place on the Azure side. Confirm with
your subscription owner before running deploys:

- An active **Azure subscription** with `Owner` or `Contributor` + `User Access
  Administrator` role on it (the scripts create role assignments).
- **`Microsoft.App`**, **`Microsoft.CognitiveServices`**, **`Microsoft.Search`**,
  **`Microsoft.Network`**, **`Microsoft.Storage`**, **`Microsoft.OperationalInsights`**,
  **`Microsoft.Insights`**, **`Microsoft.KeyVault`** resource providers registered.
- **Quota** for: Flex Consumption Function App (vCPU + memory), Azure AI Search
  Standard SKU, Azure OpenAI / Foundry model capacity in your target region.
- An **App Registration** (or Sites.Selected SPN) with the Microsoft Graph
  permissions listed in [`deployment/sharepoint-sync-func/README.md`](deployment/sharepoint-sync-func/README.md) — at minimum `Sites.Read.All`,
  `Files.Read.All`, and (for Purview) `InformationProtectionPolicy.Read.All`.
- Outbound network access from the deployment machine to:
  - `*.azure.com`, `login.microsoftonline.com` (Azure CLI auth)
  - `api.nuget.org`, `*.nuget.org` (`dotnet restore`)
  - `*.azurewebsites.net` (`func azure functionapp publish` SCM endpoint)
  - `mcr.microsoft.com`, `*.data.mcr.microsoft.com` (Function/ACA base images)
  - `github.com` (clone / pull)

If your machine sits behind a corporate proxy, set `HTTPS_PROXY`,
`HTTP_PROXY`, `NO_PROXY` and re-export them in every shell session before
running the scripts.

---

## D. One-time setup steps inside this repository

After cloning the repo:

```bash
# Inside WSL/Ubuntu OR Git Bash, at repo root
git clone git@github.com:eli-tectika/Azure-Foundry-to-SharePoint---Full-System.git
cd Azure-Foundry-to-SharePoint---Full-System

# Restore + build the Function App project once to populate the NuGet cache
cd deployment/sharepoint-sync-func
dotnet restore
dotnet build -c Release
cd ../..

# Restore + build the agent tool (requires net10.0 SDK)
cd agent-tool
dotnet restore
dotnet build -c Release
cd ..

# Authenticate to Azure
az login
az account set --subscription "<your-subscription-id>"

# Make deploy scripts executable (only needed in Git Bash; WSL preserves +x)
chmod +x deployment/*.sh deployment/sharepoint-sync-func/deploy/*.sh bicep/*.sh
```

Copy the `.env.example` files and fill in real values:

```bash
cp deployment/hub.env.example                            deployment/hub.env
cp deployment/spoke.env.example                          deployment/spoke.env
cp deployment/sharepoint-sync.env.example                deployment/sharepoint-sync.env
cp deployment/sharepoint-sync-func/.env.example          deployment/sharepoint-sync-func/.env
cp deployment/sharepoint-sync-func/local.settings.json.example \
   deployment/sharepoint-sync-func/local.settings.json
# (then edit each file in your editor)
```

---

## E. Verification

Open a fresh shell (WSL Ubuntu or Git Bash) and run:

```bash
echo "--- Versions ---"
git --version
az --version | head -3
dotnet --list-sdks
func --version
docker --version 2>/dev/null || echo "(docker optional — skipped)"
python3 --version
jq --version
zip --version | head -1
curl --version | head -1

echo "--- Azure login ---"
az account show -o table

echo "--- Azure CLI extensions ---"
az extension list -o table

echo "--- Build the Function project ---"
( cd deployment/sharepoint-sync-func && dotnet build -c Release --no-restore )

echo "--- Audit Function project for vulns/deprecation ---"
( cd deployment/sharepoint-sync-func \
    && dotnet list package --vulnerable --include-transitive \
    && dotnet list package --deprecated --include-transitive )
```

Expected results:
- `git`, `az`, `dotnet --list-sdks` shows **10.0.x**, `func`,
  `python3`, `jq`, `zip`, `curl` all return versions.
- `az account show` lists your active subscription.
- `az extension list` includes `containerapp`.
- `dotnet build` finishes with **0 Warning(s) 0 Error(s)**.
- `dotnet list package --vulnerable` and `--deprecated` both report **no
  vulnerable / no deprecated packages** (run after `dotnet restore`).

If everything above passes, the machine is ready. Run the deploy scripts in
this order:

```bash
# 1. Hub network + firewall + DNS zones
./deployment/1-deploy-hub.sh

# 2. Spoke (Foundry, AI Search, Function Storage, etc.)
./deployment/2-deploy-spoke.sh

# 3. SharePoint sync pipeline (Function App + AI Search index/skillset/agent)
./deployment/3-deploy-sharepoint-sync.sh
```

Or to deploy only the sync project to existing Azure resources:
```bash
cd deployment/sharepoint-sync-func/deploy
TARGET=func ./deploy-existing.sh
```

---

## F. Software inventory (cheat sheet)

| Tool | Version pin | Why |
|------|-------------|-----|
| Windows 10/11 64-bit | — | Host OS |
| WSL2 + Ubuntu 24.04 *(Option A)* | latest | Linux env for bash deploy scripts |
| Git Bash *(Option B alt.)* | latest | Bash on native Windows |
| `git` | latest | Clone / push |
| `gh` (optional) | latest | GitHub PRs/issues |
| **.NET SDK 10.0** | LTS, latest patch | Builds both `SharePointSyncFunc.csproj` and `agent-tool/AgentTool.csproj` |
| **Azure CLI** (`az`) | 2.66+ | All `az` commands in deploy scripts |
| `az` extension `containerapp` | latest | `az containerapp env/job ...` |
| `az` extension `application-insights` | latest | Optional AI queries from scripts |
| **Azure Functions Core Tools** (`func`) | v4 latest | `func azure functionapp publish` |
| Python 3 | 3.10+ | JSON-parsing helpers in deploy scripts |
| `jq` | 1.6+ | JSON parsing in `bicep/*.sh` |
| `zip` / `unzip` | any | `dotnet publish` artifact packaging |
| `curl` | any | SCM `/api/publish` upload |
| **Docker Desktop** *(only for ACA path)* | latest | `az acr build` / Dockerfile |
| (any text editor — VS Code recommended) | — | Editing `.env` files |

The NuGet packages used by the .NET projects are pinned in their respective
`.csproj` files. Both projects pass `dotnet list package --vulnerable
--include-transitive` and `--deprecated --include-transitive` with **zero
findings**. The only preview packages in use are
`Azure.AI.Projects` 2.0.0-beta.1, `Azure.AI.Projects.OpenAI` 2.0.0-beta.1, and
`Azure.Search.Documents` 11.8.0-beta.1 in `agent-tool/` — these are required
because the Foundry v2 agents API + agentic retrieval features have no GA
release yet.
