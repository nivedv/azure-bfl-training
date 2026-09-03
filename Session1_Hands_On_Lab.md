# Session 1 — Hands-On Lab
## Build and Explore Your Azure Resource Footprint
*Self-paced · ~45-50 minutes · Individual or pairs*
*BFL Azure & AI Foundry Training Program*

---

## Before You Start
- **Role required:** Contributor at the subscription level
- **Region:** East US or Central US only
- **Naming:** Every resource you create uses the `trainee_` prefix (storage account names: `trainee` + your initials, lowercase, no underscore — storage account names only allow lowercase letters and numbers)
- **Tool:** Azure Cloud Shell (Bash) — open it now from the Portal toolbar before continuing
- This lab is **different from today's trainer demo** — it goes further, letting you compare redundancy options and explore governance hands-on rather than just watching. Expect to spend real time here; don't rush.
- ✅ boxes mark checkpoints — do not move to the next part until you can check the box honestly.

---

## Part 1 — Explore Azure's Global Infrastructure (8 min)

Before creating anything, look at the raw infrastructure data behind everything in today's slides.

```bash
# List all regions your subscription can deploy into
az account list-locations --output table

# Check whether a specific region supports Availability Zones
az vm list-skus --location eastus --zone --size Standard_B1s --output table
```

**Try it yourself:** Run the second command against `centralus` as well. Compare the two outputs.

**Checkpoint ✅**
- [ ] I can name two regions from the list that are physically close to India (for a production deployment) even though we're using US regions in this lab
- [ ] I can explain, in one sentence, what the `--zone` flag in the second command is actually checking for

**Troubleshooting:** If `az vm list-skus` returns an empty table, double check the `--location` spelling — it must be the region's short name (`eastus`, not `East US`).

---

## Part 2 — Create Your Resource Group (5 min)

```bash
az group create \
  --name trainee_rg_lab1 \
  --location eastus \
  --tags Environment=Training Owner=<your-name> CostCenter=BFL-Training AutoDestroy=Yes
```

Replace `<your-name>` with your actual name — no spaces, use a hyphen if needed.

**Checkpoint ✅**
- [ ] Running `az group show --name trainee_rg_lab1` returns your resource group with the tags you set
- [ ] I can state which ARM scope this sits directly beneath (Subscription)

---

## Part 3 — Compare Storage Redundancy Hands-On (12 min)

Create **two** storage accounts with different redundancy tiers so you can see the difference in the Portal, not just read about it.

```bash
# Account 1: Locally Redundant Storage
az storage account create \
  --name traineelrs<yourinitials> \
  --resource-group trainee_rg_lab1 \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# Account 2: Zone-Redundant Storage
az storage account create \
  --name traineezrs<yourinitials> \
  --resource-group trainee_rg_lab1 \
  --location eastus \
  --sku Standard_ZRS \
  --kind StorageV2
```

Now in the **Portal**: open each storage account → **Data management → Redundancy**. Look at the description Azure itself gives for each tier.

**Checkpoint ✅**
- [ ] I can point to the exact setting in the Portal that shows the redundancy tier
- [ ] I can explain, in my own words, why the ZRS account is a better fit for the loan-document scenario from today's trainer demo than the LRS account
- [ ] I priced both — open **Cost Management + Billing → Pricing calculator** in a new tab and note roughly how much more ZRS costs per GB than LRS for Hot tier

**Troubleshooting:** Storage account names must be globally unique across *all* of Azure. If you get a "name already taken" error, add a number to the end (e.g. `traineelrsnv2`).

---

## Part 4 — Deploy a Zone-Pinned Virtual Machine (10 min)

```bash
az vm create \
  --resource-group trainee_rg_lab1 \
  --name trainee_vm_lab1 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --zone 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --tags Environment=Training Owner=<your-name> CostCenter=BFL-Training AutoDestroy=Yes
```

Note we used **Zone 2** here — deliberately different from the trainer demo's Zone 1, so you can see in the Portal that these are independent placements.

**Checkpoint ✅**
- [ ] In the Portal, under the VM's **Overview**, I can see which Availability Zone it landed in
- [ ] I can explain why pinning to one zone alone is *not* the same as a zone-redundant, highly available design

**Troubleshooting:** VM creation typically takes 2–4 minutes. If it fails with a quota error, tell your trainer immediately — this is a shared training subscription and quota may need adjusting.

---

## Part 5 — Apply and Test a Resource Lock (7 min)

```bash
az lock create \
  --name protect-lab1-storage \
  --resource-group trainee_rg_lab1 \
  --resource-name traineezrs<yourinitials> \
  --resource-type Microsoft.Storage/storageAccounts \
  --lock-type CanNotDelete
```

**Now try to break it on purpose:**
```bash
az storage account delete \
  --name traineezrs<yourinitials> \
  --resource-group trainee_rg_lab1 \
  --yes
```

**Checkpoint ✅**
- [ ] The delete command **failed** with a lock-related error — take a screenshot of this error, it's proof the lock is doing its job
- [ ] I can explain why this failure happens *even though* I have Contributor rights on the whole resource group

---

## Part 6 — Verify Everything Through the Portal (5 min)

1. **Resource Graph Explorer** → run:
   ```kusto
   Resources
   | where resourceGroup == "trainee_rg_lab1"
   | project name, type, location, tags
   ```
   Confirm all four resources (2 storage accounts, 1 VM, plus its supporting networking resources) appear.
2. **Cost Management + Billing → Cost analysis** → filter to `trainee_rg_lab1` and note the running cost so far.

**Checkpoint ✅**
- [ ] My Resource Graph query returned results without errors
- [ ] I can name one resource in the group I didn't explicitly create myself (hint: the VM creates supporting resources like a NIC and disk automatically)

---

## Part 7 — Clean Up

```bash
# Remove the lock first — the resource group cannot be deleted while it's protected
az lock delete \
  --name protect-lab1-storage \
  --resource-group trainee_rg_lab1 \
  --resource-name traineezrs<yourinitials> \
  --resource-type Microsoft.Storage/storageAccounts

# Delete everything in one action
az group delete --name trainee_rg_lab1 --yes --no-wait
```

**Final Checkpoint ✅**
- [ ] I understand that deleting the resource group removed the VM, both storage accounts, and every supporting resource in one action — this is the "shared lifecycle" concept from Session 1 in practice

---

## Self-Assessment
If every checkbox above is ticked, you've hands-on touched every major concept from today: regions and zones, resource groups, storage redundancy tradeoffs, zone-pinned compute, resource locks, and Portal-based verification tools.

**Still stuck on something?** Check the Session 1 FAQ document first, then flag your trainer.

---
*Prepared by Nived Varma. Reference: Microsoft Learn (learn.microsoft.com/azure) for current CLI syntax and pricing.*
