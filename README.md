<div align="center">
  
# Automated DVWA Security Testing App
 
**A Python application that automates security testing in Microsoft Azure**
 
`Microsoft Azure` · `Terraform` · `Packer` · `Ansible` · `Python + Tkinter` · `Kali Linux` · `Windows Server 2025` · `DVWA`
 
[About](#about) •
[Architecture](#architecture) •
[Features](#features) •
[Installation](#installation) •
[Usage](#usage) •
[Troubleshooting](#troubleshooting)
 
</div>

> [!CAUTION]
> **Educational project.** DVWA is an **intentionally vulnerable** web application. Use ADSTA only in the isolated environment created by this project, in your own Azure subscription, and only against your own resources. The Windows machines have no public IP, and access to Kali is allowed only from your public IP. Do not expose DVWA to the Internet.
 
> [!NOTE]
> The application's interface and the Ansible task names are in **Romanian**. This guide quotes the labels exactly as they appear on screen, followed by an English translation in parentheses.
 
<a name="about"></a>
 
## 📖 About the project
 
This project builds an **isolated, reproducible** test environment in **Microsoft Azure** with Terraform, Packer and Ansible. The **ADSTA** application runs on the Kali Linux machine and compares how an unprotected system and a protected system respond to the same tests.

The environment has **three virtual machines**: a **Kali Linux** attack machine that runs the ADSTA application, and two **Windows Server 2025** targets that host the same vulnerable web application, **DVWA**, configured differently:
 
- **Vulnerable Windows**: firewall disabled, DVWA security level `low`, all traffic from the Kali subnet allowed;
- **Protected Windows**: firewall enabled, DVWA security level `impossible`, only the required ports allowed.

| Feature | Description |
|---|---|
| 🔁 **Automated infrastructure** | Terraform creates the Azure resources, Packer prepares the Kali image, and Ansible configures the Windows targets. |
| 🔒 **Isolated environment** | A dedicated virtual network, one subnet per machine, and different NSG rules for each target. |
| ⚖️ **Side-by-side comparison** | The same test runs on both targets, and the results are shown next to each other. |
| 🖱️ **Everything from one interface** | Target configuration and testing are done from the ADSTA application, with no manual commands on the Windows machines. |
 
<a name="architecture"></a>
 
## 🏗️ Architecture
 
```text
    Your workstation (your public IP, detected automatically by Terraform)
                     │
                     │ RDP · TCP 3389 · allowed only from your public IP
                     │
┌────────────────────┼───────────────────────────────────────────────────────────────┐
│                    ▼                                                               │
│   ┌─ snet-kali · 10.0.1.0/24 ───────┐      Microsoft Azure                         │
│   │  Kali Linux + ADSTA app         │      Resource group:  rg-<project_name>      │
│   │  10.0.1.10 · only public IP     │      Virtual network: 10.0.0.0/16            │
│   └────────────────┬────────────────┘                                              │
│                    │       only TCP 80, 443, 3389, 5985                            │
│                    ├─────────────────────────────────────────┐                     │
│                    │ all traffic                             │                     │
│                    │                                         │                     │
│                    ▼                                         ▼                     │
│   ┌─ snet-vuln · 10.0.2.0/24 ───────┐       ┌─ snet-protd · 10.0.3.0/24 ──────┐    │
│   │  Vulnerable Windows             │       │  Protected Windows              │    │
│   │  10.0.2.10                      │       │  10.0.3.10                      │    │
│   │  Firewall OFF · DVWA low        │       │  Firewall ON · DVWA impossible  │    │
│   └─────────────────────────────────┘       └─────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────────────┘
```
 
| | Vulnerable Windows | Protected Windows |
|---|---|---|
| Private IP | `10.0.2.10` | `10.0.3.10` |
| Public IP | no | no |
| Traffic allowed from the Kali subnet (NSG) | all traffic | only TCP `3389`, `80`, `443`, `5985`; everything else is blocked |
| Windows Firewall | disabled (Domain, Private and Public profiles) | enabled, with the "Allow Apache HTTP" rule (TCP 80) |
| DVWA security level | `low` | `impossible` |
 
**How the environment is built and used:**
 
```text
1. terraform/1_kali_image_RG            →  the kali-image-rg resource group
2. packer/kali_image                    →  the kali-image image
3. terraform/2_project_infrastructure   →  the network and the 3 virtual machines
4. ADSTA · "Configurare Windows"        →  Ansible configures the targets over WinRM
5. ADSTA · tests and attacks            →  results shown side by side
```
 
<a name="features"></a>
 
## ✨ Features
 
### Infrastructure as code
 
- **Terraform stage 1** creates the `kali-image-rg` resource group, where the Kali image is stored.
- **Packer** builds the `kali-image` image from Kali Linux in the Azure Marketplace: XFCE desktop with RDP access (xrdp), Ansible and the Windows collections, network tools, and the ADSTA application preinstalled, with a desktop icon.
- **Terraform stage 2** creates 20 resources: the virtual network, 3 subnets, 3 NSGs and their associations, the public IP, 3 network interfaces, 3 virtual machines, and the extensions that enable WinRM on the Windows targets.
- The RDP rule for Kali is automatically restricted to **your public IP**, detected while Terraform runs.
### Automated target configuration (Ansible)
 
A single playbook, run from Kali over WinRM, prepares both Windows machines in 9 stages: WinRM connection check → firewall → Chocolatey → XAMPP → Apache and MySQL services → DVWA download → DVWA configuration (security level) → database → DVWA tables. Each step first checks whether it is already done, so the playbook can be run several times.
 
### The ADSTA application
 
- **Configuration screen:** starts the playbook and shows, in real time, the state of each of the 9 stages (⚪ waiting · 🔵 in progress · 🟢 done · 🔴 error), the Ansible output with explanations, and the total duration.
- **Testing screen:** runs the tests on the vulnerable machine, the protected machine, or both in parallel, and shows the results side by side.

| Category | Test | What it checks |
|---|---|---|
| System | **PING** | Whether the machine replies to ICMP requests. |
| System | **NMAP** | The ports and services visible from the network. |
| System | **NETCAT** | TCP connectivity to a fixed set of ports. |
| DVWA | **Brute Force** | Whether the login accepts repeated automated attempts. |
| DVWA | **Command Injection** | Whether the page runs additional system commands. |
| DVWA | **SQL Injection** | Whether the database query can be altered to return extra data. |
| DVWA | **XSS Reflected** | Whether submitted text is reflected unfiltered in the page. |
 
Each DVWA test ends with a **VULNERABIL** (*vulnerable*) or **PROTEJAT** (*protected*) verdict and an explanation. The **Încearcă!** (*Try it!*) button opens an RDP session to the selected target for manual checking in a browser.
 
<a name="technologies"></a>
 
## 🧰 Technologies
 
| Component | Role in the project | Version |
|---|---|---|
| Microsoft Azure | Hosts the whole infrastructure | — |
| Azure CLI | Authentication (`az login`) and accepting the Marketplace terms | — |
| Terraform | Infrastructure stages 1 and 2 | 1.14.9 · `azurerm` 4.69.0 · `http` 3.5.0 |
| Packer | Custom Kali Linux image | 1.15.3 · `azure` plugin ~> 2 |
| Ansible | Configuration of the Windows targets over WinRM | `ansible.windows`, `chocolatey.chocolatey`, `community.windows` collections |
| Python 3 + Tkinter | The ADSTA application | — |
| Kali Linux | Attack machine | Marketplace `kali-2026-1` |
| Windows Server | Targets | 2025 Datacenter (`2025-datacenter-g2`) |
| Chocolatey + XAMPP | Installs Apache, MySQL and PHP on Windows | `xampp-81` package |
| DVWA | Target web application | `master` branch of `digininja/DVWA` |
 
<a name="prerequisites"></a>
 
## ✅ Prerequisites
 
- A **Microsoft Azure** account with an active subscription and quota for at least **6 vCPUs** (3 × `Standard_B2s_v2` machines with 2 vCPUs each). The thesis used an *Azure for Students* subscription.
- On your workstation: **Azure CLI**, **Terraform**, **Packer** and **Git**.
- An **RDP** client, for example *Remote Desktop Connection* on Windows.
> [!TIP]
> The commands below work in both PowerShell and Bash. Each step starts from the **repository root**.
 
<a name="installation"></a>
 
## 🚀 Installation
 
### Step 1 — Clone the repository
 
```bash
git clone https://github.com/ovidiuoros/adsta-azure-security-testing.git
cd adsta-azure-security-testing
```
 
### Step 2 — Sign in to Azure
 
```bash
az login
```
 
If you have several subscriptions, select the one where you will create the environment (the same as `subscription_id` in the configuration files):
 
```bash
az account set --subscription "<subscription_id>"
```
 
### Step 3 — Create the configuration files
 
The project includes `*.example` files, which you copy under their real names and then fill in:
 
```bash
cp terraform/1_kali_image_RG/terraform.tfvars.example terraform/1_kali_image_RG/terraform.tfvars
cp packer/kali_image/variables.pkrvars.hcl.example packer/kali_image/packer.auto.pkrvars.hcl
cp terraform/2_project_infrastructure/terraform.tfvars.example terraform/2_project_infrastructure/terraform.tfvars
cp ansible/windows_configurations/windows_credentials.yml.example ansible/windows_configurations/windows_credentials.yml
cp python_app/variables.py.example python_app/variables.py
```
 
| # | File created | What you fill in |
|---|---|---|
| 1 | `terraform/1_kali_image_RG/terraform.tfvars` | `subscription_id`, `location` |
| 2 | `packer/kali_image/packer.auto.pkrvars.hcl` | `subscription_id`, `location` |
| 3 | `terraform/2_project_infrastructure/terraform.tfvars` | `subscription_id`, `location`, `project_name`, `admin_username`, `admin_password`, `kali_admin_password` |
| 4 | `ansible/windows_configurations/windows_credentials.yml` | `windows_username`, `windows_password` |
| 5 | `python_app/variables.py` | the Windows credentials (the IP addresses stay as in the example) |
 
**Values that must match across files:**
 
| Value | Where it appears |
|---|---|
| Subscription and region | `subscription_id` and `location` in files **1**, **2** and **3** |
| Windows username | `admin_username` (3) = `windows_username` (4) = `windows_username` (5) |
| Windows password | `admin_password` (3) = `windows_password` (4) = `windows_password` (5) |
| Vulnerable machine IP | `10.0.2.10`: `vulnerable_private_ip` (3) and `vulnerable_vm_ip` (5); the same address is defined in `inventory.ini` |
| Protected machine IP | `10.0.3.10`: `protected_private_ip` (3) and `protected_vm_ip` (5); the same address is defined in `inventory.ini` |
 
Keep the network and IP addresses from the examples: `inventory.ini` and the ADSTA application texts use these values.
 
> [!IMPORTANT]
> Files **4** and **5** must be created **before Step 6**. Packer copies the `ansible/windows_configurations` and `python_app` folders into the Kali image exactly as they are on disk, and the application does not start without `variables.py`.
 
> [!NOTE]
> `project_name` must be at most **8 characters** long (letters and digits): the Windows machine names (`vmvuln-<project_name>`, `vmprot-<project_name>`) also become Windows computer names, which are limited to 15 characters. Azure also rejects some reserved usernames (for example `admin`, `administrator`, `root`) and requires complex passwords.
 
### Step 4 — Terraform stage 1: the image resource group
 
```bash
cd terraform/1_kali_image_RG
terraform init
terraform apply
cd ../..
```
 
✔️ Result: the `kali-image-rg` resource group. The thesis uses `terraform apply -auto-approve`, which skips the manual confirmation.
 
### Step 5 — Accept the Kali Marketplace image terms
 
The Kali image in the Azure Marketplace has a plan (`kali-2026-1`) whose terms must be accepted once for the subscription you use:
 
```bash
az vm image terms accept --publisher kali-linux --offer kali --plan kali-2026-1
```
 
### Step 6 — Build the Kali image with Packer
 
```bash
cd packer/kali_image
packer init .
packer build .
cd ../..
```
 
✔️ Result: the `kali-image` image in the `kali-image-rg` resource group. The build takes about **14 minutes**.
 
> [!IMPORTANT]
> Run Packer **from the `packer/kali_image` folder**, because the template copies `../../ansible/windows_configurations` and `../../python_app` using relative paths. Equivalent option, used in the thesis: copy the example as `variables.pkrvars.hcl` and run `packer build -var-file="variables.pkrvars.hcl" .`
 
### Step 7 — Terraform stage 2: the main infrastructure
 
```bash
cd terraform/2_project_infrastructure
terraform init
terraform apply
```
 
✔️ Result: `Apply complete!` (20 resources) and the output values you need to connect:
 
| Output | Content |
|---|---|
| `kali_public_ip` | Public IP of the Kali machine |
| `kali_admin_username` | Kali machine username |
| `kali_admin_password` | Kali machine password |
 
You can show them again at any time, from the same folder, with `terraform output`. The environment is ready; continue with [Usage](#usage).
 
<a name="usage"></a>
 
## 🖥️ Usage
 
### 1. Connect to the Kali machine
 
Open your RDP client (for example *Remote Desktop Connection*), enter the `kali_public_ip` value, and sign in with `kali_admin_username` and `kali_admin_password`. The XFCE desktop of the Kali machine opens.
 
### 2. Start ADSTA
 
Double-click the **Automated DVWA Security Testing App** icon on the desktop. On the first launch, XFCE asks you to confirm that the launcher is trusted. You can also start the application from a terminal:
 
```bash
cd /usr/share/adsta/python_app && python3 adsta.py
```
 
### 3. Configure the Windows machines
 
1. Click **Configurare Windows** (*Configure Windows*). The Ansible playbook starts, and the 9 stages in the *Status configurare* (*Configuration status*) panel change state one by one.
2. On the right, follow the Ansible output and the explanation of each stage. The first configuration takes about **8 minutes**.
3. When it finishes, the message **Configurare finalizată!** (*Configuration complete!*) and the total duration appear, and the **Pornește testele** (*Start the tests*) button becomes active.
If a stage fails, it turns red and the button shows **Eroare: reîncearcă** (*Error: retry*). The messages on the right show the cause.
 
### 4. Run the tests
 
1. Click **Pornește testele** (*Start the tests*).
2. Tick **WINDOWS VULNERABIL** (*vulnerable Windows*), **WINDOWS PROTEJAT** (*protected Windows*), or both.
3. Choose the **Sistem** (*System*) tab (PING, NMAP, NETCAT) or the **DVWA** tab (Brute Force, Command Injection, SQL Injection, XSS Reflected) and click the test you want. The **i** button explains each test.
4. Compare the results in the two panels: the vulnerable target on the left, the protected one on the right.
### 5. Check manually in a browser
 
Tick a machine and click **Încearcă!** (*Try it!*). An RDP session opens to the selected Windows machine, where DVWA starts automatically in Microsoft Edge. The default DVWA account (`admin` / `password`) is shown in the results panel.
 
| Button | Action |
|---|---|
| **Șterge rezultate** (*Clear results*) | Clears the panels of the ticked machines. |
| **Informații** (*Information*) | Describes the testing screen. |
| **Înapoi** (*Back*) | Returns to the configuration screen. To re-enable **Pornește testele** (*Start the tests*), run **Configurare Windows** (*Configure Windows*) again; stages already done are skipped. |
| **Ieșire** (*Exit*) | Closes the application. |
  
## 📁 Project structure
 
```text
adsta-azure-security-testing/
├── ansible/
│   └── windows_configurations/        # playbook for the two Windows machines
│       ├── ansible.cfg
│       ├── inventory.ini
│       ├── windows_configurations.yml
│       └── windows_credentials.yml.example
├── packer/
│   └── kali_image/                    # custom Kali Linux image
│       ├── kali.pkr.hcl
│       ├── packer_config.pkr.hcl
│       ├── variables.pkr.hcl
│       └── variables.pkrvars.hcl.example
├── python_app/                        # the ADSTA application (Python + Tkinter)
│   ├── adsta.py
│   ├── adsta.desktop
│   ├── app_icon.png
│   └── variables.py.example
├── terraform/
│   ├── 1_kali_image_RG/               # stage 1: resource group for the image
│   └── 2_project_infrastructure/      # stage 2: network and the 3 virtual machines
└── README.md
```
 
| Folder | Documentation |
|---|---|
| `ansible/` | [README](ansible/README.md) · [windows_configurations](ansible/windows_configurations/README.md) |
| `packer/` | [README](packer/README.md) · [kali_image](packer/kali_image/README.md) |
| `python_app/` | [README](python_app/README.md) |
| `terraform/` | [README](terraform/README.md) · [1_kali_image_RG](terraform/1_kali_image_RG/README.md) · [2_project_infrastructure](terraform/2_project_infrastructure/README.md) |
 
<a name="cleanup"></a>
 
## 🧹 Removing the resources
 
The virtual machines use credit for as long as they exist. When you are done, delete the resources in reverse order of creation:
 
```bash
cd terraform/2_project_infrastructure
terraform destroy
cd ../1_kali_image_RG
terraform destroy
```
 
Stage 1 deletes the `kali-image-rg` resource group together with the image created by Packer.
 
> [!CAUTION]
> `terraform.tfvars`, `packer.auto.pkrvars.hcl`, `windows_credentials.yml`, `variables.py` and the Terraform state (`terraform.tfstate`) contain sensitive data: the subscription ID and the passwords. Do not publish them in the repository.
 
<a name="troubleshooting"></a>
 
## 🛠️ Troubleshooting
 
| Symptom | Likely cause | Solution |
|---|---|---|
| Packer or Terraform reports an error about the Kali image terms or plan | The Marketplace terms were not accepted for the subscription | Run the command from [Step 5](#installation). |
| Packer cannot find `../../ansible/...` or `../../python_app` | The command was not run from `packer/kali_image` | Run `packer build .` from `packer/kali_image`. |
| Packer reports unset variables | The values file is not named `*.auto.pkrvars.hcl` | Rename it, or use `-var-file="variables.pkrvars.hcl"`. |
| Terraform rejects the computer name of the Windows machines | `project_name` is longer than 8 characters | Choose a shorter `project_name`. |
| The RDP connection to Kali fails | Your public IP has changed since the last `terraform apply` | Run `terraform apply` again in `terraform/2_project_infrastructure`. |
| The application does not open from the desktop | `variables.py` is missing from the image | Start the application from a terminal to see the error. On Kali, use `sudo` to copy `variables.py.example` to `variables.py` in `/usr/share/adsta/python_app`, then fill it in. |
| The message *„Playbook-ul nu a pornit corect…”* (*"The playbook did not start correctly…"*) | `windows_credentials.yml` or another file is missing from `/usr/share/adsta/windows_configurations` | Check the folder. If the credentials file is missing, create it with `sudo` from `windows_credentials.yml.example`, in the same folder. |
| The *Conexiune WinRM* (*WinRM connection*) stage turns red | The Windows credentials do not match, or the machines are still starting | Check the values from [Step 3](#installation) and retry. |
| DVWA tests show *„Autentificarea a eșuat…”* (*"Authentication failed…"*) | The configuration did not finish successfully, or DVWA is not responding | Run **Configurare Windows** (*Configure Windows*) again. |
 
<a name="academic"></a>
 
## 🎓 Academic context
 
Diploma project (bachelor's thesis) completed in 2026 at the **Technical University of Cluj-Napoca**, Faculty of Electronics, Telecommunications and Information Technology, Telecommunications Technologies and Systems program.
 
