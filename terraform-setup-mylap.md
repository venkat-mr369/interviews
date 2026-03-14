### Set up your Terraform project locally in VS Code

### Step 1 — Open terminal in VS Code

```
Step 1 — Open terminal in VS Code from your folder
```

---

### Step 2 — Create the folder structure

```bash
# Navigate to where you want the project (e.g. Desktop or D: drive)
cd E:\my-learning\my-terraform\amskap-proj

# Create the project folder
mkdir amskap-proj
cd amskap-proj

# Create all terraform files at once
New-Item main.tf
New-Item outputs.tf
New-Item variables.tf
```

Or if you're on **Git Bash / WSL**:
```bash
mkdir amskap-proj && cd amschap-proj
touch main.tf outputs.tf variables.tf
```

---

### Step 3 — Open the folder in VS Code

```bash
# This opens VS Code directly in the folder
code .
```

Or from VS Code: **File → Open Folder → select `amskap-proj`**

---

### Step 4 — Install required tools on Windows

```powershell
# Install Chocolatey (Windows package manager) — run as Administrator
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Terraform
choco install terraform -y

# Install Google Cloud CLI
choco install gcloudsdk -y

# Verify installations
terraform --version
gcloud --version
```

---

### Step 5 — Authenticate to GCP from your laptop

```powershell
# Login to your Google account
gcloud auth login

# Set your project
gcloud config set project ams-kap

# Generate Application Default Credentials
gcloud auth application-default login

# Impersonate your service account
gcloud config set auth/impersonate_service_account deploy@ams-kap.iam.gserviceaccount.com
```

---

### Step 6 — Your folder structure should look like this

```
amskap-proj/
│
├── main.tf          ← VPC, Subnets, VMs, Firewall
├── outputs.tf       ← Print VM IPs after deploy
└── variables.tf     ← (optional) reusable variables
```

Paste the `main.tf` from the previous answer into VS Code, then run:

```powershell
terraform init      # downloads GCP provider
terraform plan      # preview resources
terraform apply     # deploy to GCP
```

---

### Step 7 — Install VS Code Terraform extension

Press `Ctrl+Shift+X` → search **HashiCorp Terraform** → Install

This gives you syntax highlighting, autocomplete, and error checking inside `.tf` files.

---

### Quick checklist

| Step | Command | Status |
|---|---|---|
| Create folder | `mkdir amskap-proj` | |
| Auth to GCP | `gcloud auth login` | |
| Set project | `gcloud config set project ams-kap` | |
| ADC login | `gcloud auth application-default login` | |
| Init Terraform | `terraform init` | |
| Deploy | `terraform apply` | |

> If `gcloud` is not recognized after install, **restart VS Code** or open a new terminal — the PATH needs to refresh.
