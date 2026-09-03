# Participant Hands-On Lab
## Session 2 — Azure Networking, Identity & Security
### Scenario: BFL Claims Processing Microservice
### BFL Azure Cloud & Microsoft AI Foundry Training Program

**Author:** Nived Varma

---

## Scenario

BFL is piloting a **Claims Processing Microservice** that reads scanned claim documents from **Blob Storage** and writes processed results back. Unlike the trainer's demo (loan platform with a public web tier), this service is **fully internal** — it's called only by other BFL systems inside the network, never directly by customers.

Your job in this lab is to build the internal network path, secure it, and connect the microservice's identity to storage — **without** an internet-facing load balancer this time, using an **internal load balancer** instead, and validating everything yourself.

This is independent practice: work through each task, then check your work against the **Validation** step before moving on. If something doesn't match, see **Troubleshooting Hints** at the end before asking the trainer.

**Naming convention:** `trn_<your-participant-id>-<resource-name>`, except where Azure naming rules force a variant (noted inline).

---

## Learning Objectives

By completing this lab you will independently:

1. Build an isolated VNet/subnet with an NSG that allows traffic **only from a named source subnet**, not the internet.
2. Deploy an **internal** Standard Load Balancer (no public IP).
3. Create a Storage account and lock it down at the network level.
4. Attach a system-assigned managed identity to a VM and grant it **Storage Blob Data Reader** via RBAC.
5. Verify — using Azure tools, not guesswork — that access works exactly as intended and nothing more.

---

## Task 1 — Resource Group and VNet

**Objective:** Create an isolated environment for the claims microservice.

**CLI (Azure CLI):**
```bash
az group create --name trn_<your-participant-id>-rg-claims --location eastus

az network vnet create \
  --name trn_<your-participant-id>-claims-vnet \
  --resource-group trn_<your-participant-id>-rg-claims \
  --location eastus \
  --address-prefix 10.30.0.0/16 \
  --subnet-name svc-subnet \
  --subnet-prefix 10.30.1.0/24
```

**Portal steps:**
1. **Resource groups** → **+ Create** → Name `trn_<your-participant-id>-rg-claims` → Region **East US** → **Create**.
2. **Virtual networks** → **+ Create** → RG `trn_<your-participant-id>-rg-claims` → Name `trn_<your-participant-id>-claims-vnet` → Region **East US**.
3. **IP Addresses** → address space `10.30.0.0/16` → default subnet: rename to `svc-subnet`, range `10.30.1.0/24` → **Review + create** → **Create**.

**Validation:** `az network vnet show --name trn_<your-participant-id>-claims-vnet --resource-group trn_<your-participant-id>-rg-claims --query "subnets[].name"` should return `["svc-subnet"]`.

---

## Task 2 — NSG Restricted to a Named Source Subnet

**Objective:** The `svc-subnet` should accept traffic only from an "internal callers" subnet (`10.30.2.0/24`) on port 443 — no internet access at all.

**CLI:**
```bash
az network vnet subnet create \
  --name callers-subnet \
  --resource-group trn_<your-participant-id>-rg-claims \
  --vnet-name trn_<your-participant-id>-claims-vnet \
  --address-prefix 10.30.2.0/24

az network nsg create \
  --name nsg-svc \
  --resource-group trn_<your-participant-id>-rg-claims \
  --location eastus

az network nsg rule create \
  --nsg-name nsg-svc \
  --resource-group trn_<your-participant-id>-rg-claims \
  --name Allow-Callers-HTTPS \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.30.2.0/24 \
  --destination-port-ranges 443

az network vnet subnet update \
  --name svc-subnet \
  --resource-group trn_<your-participant-id>-rg-claims \
  --vnet-name trn_<your-participant-id>-claims-vnet \
  --network-security-group nsg-svc
```

**Portal steps:**
1. Open `trn_<your-participant-id>-claims-vnet` → **Subnets** → **+ Subnet** → Name `callers-subnet` → range `10.30.2.0/24` → **Add**.
2. **Network security groups** → **+ Create** → RG `trn_<your-participant-id>-rg-claims` → Name `nsg-svc` → **Create**.
3. Open `nsg-svc` → **Inbound security rules** → **+ Add** → Source **IP Addresses** `10.30.2.0/24` → Destination port `443` → Protocol **TCP** → Action **Allow** → Priority `100` → Name `Allow-Callers-HTTPS` → **Add**.
4. `trn_<your-participant-id>-claims-vnet` → **Subnets** → `svc-subnet` → NSG = `nsg-svc` → **Save**.

**Validation:** In `nsg-svc` → **Effective security rules** (once associated to a NIC) or simply confirm in **Inbound security rules** that the default `AllowInternet` rule sits **below** your custom rule at a lower priority number, and that no rule references `Internet` as an allowed source.

**Checkpoint question:** Why is `10.30.2.0/24` used as the source instead of `Internet` or `VirtualNetwork`? *(Answer: the microservice must only be reachable from the specific internal-caller subnet — using `VirtualNetwork` would also allow the data/other subnets, which is broader than required.)*

---

## Task 3 — Internal Load Balancer

**Objective:** Unlike the trainer's public-facing LB, this microservice is called only from inside the network — deploy an **internal** Standard Load Balancer with no public IP.

**CLI:**
```bash
az network lb create \
  --name trn_<your-participant-id>-ilb-claims \
  --resource-group trn_<your-participant-id>-rg-claims \
  --sku Standard \
  --vnet-name trn_<your-participant-id>-claims-vnet \
  --subnet svc-subnet \
  --frontend-ip-name fe-claims \
  --backend-pool-name be-claims \
  --private-ip-address-version IPv4

az network lb probe create \
  --lb-name trn_<your-participant-id>-ilb-claims \
  --resource-group trn_<your-participant-id>-rg-claims \
  --name https-probe --protocol Tcp --port 443

az network lb rule create \
  --lb-name trn_<your-participant-id>-ilb-claims \
  --resource-group trn_<your-participant-id>-rg-claims \
  --name lb-rule-https \
  --protocol Tcp --frontend-port 443 --backend-port 443 \
  --frontend-ip-name fe-claims --backend-pool-name be-claims --probe-name https-probe
```

**Portal steps:**
1. **Load balancers** → **+ Create** → RG `trn_<your-participant-id>-rg-claims` → Name `trn_<your-participant-id>-ilb-claims` → SKU **Standard** → Type **Internal**.
2. **Frontend IP configuration** → **+ Add** → Name `fe-claims` → Virtual network `trn_<your-participant-id>-claims-vnet` → Subnet `svc-subnet` → Assignment **Dynamic** → **Add**.
3. **Backend pools** → **+ Add** → Name `be-claims` → associate to `svc-subnet` → **Add**.
4. **Health probes** → **+ Add** → Name `https-probe` → TCP `443` → **Add**.
5. **Load balancing rules** → **+ Add** → Name `lb-rule-https` → Frontend `fe-claims` → Backend `be-claims` → TCP `443` → Probe `https-probe` → **Add**.
6. **Review + create** → **Create**.

**Validation:** Open the load balancer's **Overview** — confirm the **Type** shows **Internal** and there is **no** Public IP address listed anywhere in the resource.

**Checkpoint question:** What would happen to this design if you'd accidentally picked **Public** instead of **Internal**? *(Answer: the LB would provision a public IP and be reachable from the internet by default, defeating the "internal only" requirement — always double-check the Type field before Review + create.)*

---

## Task 4 — Storage Account Locked to the VNet

**Objective:** Create the Blob Storage account the microservice reads/writes, and restrict network access to only the claims VNet.

**CLI:**
```bash
az storage account create \
  --name trnclaimsstg<your-participant-id> \
  --resource-group trn_<your-participant-id>-rg-claims \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

az storage account network-rule add \
  --account-name trnclaimsstg<your-participant-id> \
  --resource-group trn_<your-participant-id>-rg-claims \
  --vnet-name trn_<your-participant-id>-claims-vnet \
  --subnet svc-subnet

az storage account update \
  --name trnclaimsstg<your-participant-id> \
  --resource-group trn_<your-participant-id>-rg-claims \
  --default-action Deny
```

**Portal steps:**
1. **Storage accounts** → **+ Create** → RG `trn_<your-participant-id>-rg-claims` → Name `trnclaimsstg<your-participant-id>` (lowercase, no hyphens/underscores — storage account names allow letters/numbers only) → Region **East US** → Redundancy **LRS** → **Review + create** → **Create**.
2. Open the account → **Networking** → **Enabled from selected virtual networks and IP addresses** → **+ Add existing virtual network** → select `trn_<your-participant-id>-claims-vnet` / `svc-subnet` → **Add** → **Save**.

**Validation:** Try browsing to the storage account's Blob endpoint from a browser outside the VNet — it should be denied. `az storage account show --name trnclaimsstg<your-participant-id> --query networkRuleSet.defaultAction` should return `"Deny"`.

**Note:** Storage account names are globally unique, lowercase, and cannot contain hyphens/underscores — hence the compressed naming (`trnclaimsstg<id>`) rather than the standard `trn_<id>-` pattern.

---

## Task 5 — Managed Identity + RBAC on Storage

**Objective:** A VM representing the microservice (assume `trn_<your-participant-id>-vm-claims` already exists from an earlier compute lab) reads blobs using its own identity — no storage account key anywhere.

**CLI:**
```bash
az vm identity assign \
  --name trn_<your-participant-id>-vm-claims \
  --resource-group trn_<your-participant-id>-rg-claims

PRINCIPAL_ID=$(az vm show --name trn_<your-participant-id>-vm-claims \
  --resource-group trn_<your-participant-id>-rg-claims --query identity.principalId -o tsv)

STORAGE_ID=$(az storage account show --name trnclaimsstg<your-participant-id> \
  --resource-group trn_<your-participant-id>-rg-claims --query id -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID
```

**Portal steps:**
1. Open `trn_<your-participant-id>-vm-claims` → **Identity** → **System assigned** → **On** → **Save**.
2. Open `trnclaimsstg<your-participant-id>` → **Access control (IAM)** → **+ Add** → **Add role assignment** → Role **Storage Blob Data Reader** → Members → **Managed identity** → select your VM → **Review + assign**.

**Validation:** From the VM (if you have console/SSH access), run `az login --identity` followed by `az storage blob list --account-name trnclaimsstg<your-participant-id> --container-name <any-container> --auth-mode login` — it should succeed with **no key or connection string supplied anywhere**.

**Checkpoint question:** If this role were assigned as **Storage Blob Data Contributor** instead of **Reader**, what changes? *(Answer: the identity could also write/delete blobs — always confirm the microservice genuinely needs write access before granting it; Reader is correct for a read-only claims viewer, Contributor if it also writes processed results back.)*

---

## Task 6 — Enable Defender for Storage

**Objective:** Turn on continuous security monitoring for the storage account you just locked down.

**CLI:**
```bash
az security pricing create --name StorageAccounts --tier Standard
```

**Portal steps:**
1. **Microsoft Defender for Cloud** → **Environment settings** → your subscription → **Defender plans** → toggle **Storage** → **On** → **Save**.
2. **Overview** → **Recommendations** → filter by resource type **Storage account** → confirm `trnclaimsstg<your-participant-id>` now appears in the assessed resource list (may take a few minutes to populate).

**Validation:** The storage account should now appear under **Inventory** in Defender for Cloud with a Defender plan status of **On**.

---

## Task 7 — Clean-up

```bash
az group delete --name trn_<your-participant-id>-rg-claims --yes --no-wait
```

**Portal steps:**
1. **Resource groups** → open `trn_<your-participant-id>-rg-claims` → **Delete resource group** → type the name to confirm → **Delete**.

---

## Troubleshooting Hints

| Symptom | Likely Cause |
|---|---|
| Storage account creation fails with a naming error | Name must be 3–24 characters, lowercase letters and numbers only — no hyphens or underscores. |
| Role assignment command succeeds but access still denied | RBAC role assignments can take a few minutes to propagate; also confirm you assigned the role at the **storage account** scope, not a different resource. |
| Internal Load Balancer shows a public IP | You likely selected **Public** instead of **Internal** for Type at creation — this can't be changed after the fact; delete and recreate. |
| `az login --identity` fails on the VM | Managed identity may not have finished provisioning — wait 1–2 minutes after enabling it, or confirm the VM has network access to Azure's Instance Metadata Service (`169.254.169.254`) — this should never be blocked by an NSG. |
| NSG rule seems to have no effect | Confirm the NSG is associated to the **subnet** (Task 2) — creating the rule alone does nothing until the NSG itself is linked to `svc-subnet`. |

---

*Reference: Microsoft Learn — Azure Load Balancer (internal), Azure Storage network security, Azure RBAC built-in roles, Managed identities for Azure resources (learn.microsoft.com).*
