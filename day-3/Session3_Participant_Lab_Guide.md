# Session 3 — Participant Lab Guide
### Triggering Governance Guardrails via the Azure Portal
**Program:** BFL Azure & AI Foundry Training | **Author:** Nived Varma

**Before you start:**
- Sign in at portal.azure.com with your assigned participant account
- Confirm your resource group: `trn_<your-participant-id>_rg`
- Replace `<id>` in every resource name below with your participant number (e.g., `07`)
- Do **not** use the shared `trainer_shared_rg` resource group except where an exercise tells you to

---

## Exercise 1 — Allowed Locations (Deny)

1. In the Portal search bar, type **Storage accounts** and select it
2. Click **+ Create**
3. **Basics** tab:
   - Subscription: (the shared training subscription)
   - Resource group: `trn_<id>_rg`
   - Storage account name: `trn<id>loctest`
   - Region: **West Europe**
   - Performance: Standard
   - Redundancy: Locally-redundant storage (LRS)
4. Click **Review**
5. Click **Create**
6. **Expected result:** deployment fails. Click **Deployment details** / **Operation details** to expand the error
7. In the error box, locate the line containing `RequestDisallowedByPolicy` and the policy assignment name
8. **Write down** the exact policy assignment name shown in your error — you'll need it for the lab worksheet

---

## Exercise 2 — Allowed VM SKUs (Deny)

1. In the Portal search bar, type **Virtual machines** and select it
2. Click **+ Create** → **Azure virtual machine**
3. **Basics** tab:
   - Resource group: `trn_<id>_rg`
   - Virtual machine name: `trn<id>vmtest`
   - Region: **East US**
   - Image: Ubuntu Server (any current LTS)
   - Size: click **See all sizes** → select **Standard_D2s_v3** (deliberately not B1s)
   - Authentication: password or SSH key, your choice
4. Click **Review + create**
5. Click **Create**
6. **Expected result:** deployment fails, error references the allowed VM SKUs policy
7. **Do not retry with a different disallowed size** — move to the next step once you've captured the error

---

## Exercise 3 — Require Tag (Append / Modify — non-blocking)

1. In the Portal search bar, type **Storage accounts** and select it
2. Click **+ Create**
3. **Basics** tab:
   - Resource group: `trn_<id>_rg`
   - Storage account name: `trn<id>tagtest`
   - Region: **East US**
   - Performance: Standard
   - Redundancy: LRS
4. Click the **Tags** tab
5. Leave the tag fields **empty** — do not add a CostCenter tag yourself
6. Click **Review + create** → **Create**
7. **Expected result:** deployment **succeeds** this time
8. Once deployment finishes, click **Go to resource**
9. In the left menu, click **Tags**
10. **Confirm:** you should see `CostCenter = BFL-Training-Default` even though you never entered it

---

## Exercise 4 — Resource Lock

1. In the Portal search bar, type **Resource groups** and select it
2. Click on **trainer_shared_rg** (this is the shared resource group — you have read access only)
3. Open the storage account inside it
4. In the left menu, click **Delete** (top toolbar) or attempt to delete the resource
5. **Expected result:** an error appears stating the resource cannot be deleted because it has a lock (`ScopeLocked`)
6. Click **Locks** in the left menu of the resource to view the lock details (name, type) — you will **not** be able to remove it
7. **Compare** this error message to the one from Exercise 1 — note it does **not** mention a policy assignment name

---

## Exercise 5 (Optional) — Not Allowed Resource Types

1. In the Portal search bar, type **Public IP addresses** and select it
2. Click **+ Create**
3. **Basics** tab:
   - Resource group: `trn_<id>_rg`
   - Name: `trn<id>pip`
   - Region: **East US**
   - IP Version: IPv4
   - SKU: Standard
4. Click **Review + create** → **Create**
5. **Expected result:** deployment fails, error references the "Not allowed resource types" policy assignment
6. **Note:** this is a Deny effect, same family as Exercises 1 and 2, but on a different resource type

---

## Exercise 6 (Optional) — Activity Log to Log Analytics (Observe Only)

This guardrail is subscription-wide and was already remediated before the session — there is nothing for you to trigger individually. Instead, observe it:

1. In the Portal search bar, type **Log Analytics workspaces** and select it
2. Click **trainer-session3-law**
3. In the left menu, click **Logs**
4. If a query window opens with suggestions, click **Close**
5. In the query editor, type: `AzureActivity | take 50`
6. Click **Run**
7. **Observe:** subscription activity log entries appear here automatically — this is the result of the DeployIfNotExists policy that ran before the session started
8. Back on the workspace's **Overview** page, in the left menu click **Policy** (or navigate to **Policy → Compliance** at the subscription level) and locate `trainer-activitylog-to-law` — confirm its compliance state shows **Compliant**

---

## Lab Worksheet — Fill In As You Go

| Exercise | Policy Assignment Name (from error) | Effect Observed | Blocked or Corrected? |
|---|---|---|---|
| 1 — Locations | | | |
| 2 — VM SKUs | | | |
| 3 — Tag | | | |
| 4 — Lock | (no policy name — lock, not policy) | | |
| 5 — Resource Types | | | |
| 6 — Activity Log | (no direct trigger — observed only) | DeployIfNotExists | Neither — auto-created |

---

## Cleanup (Do This Before the Session Ends)

Delete the test resources you created in Exercises 1–3 and 5 (skip Exercise 4/6 — those live in the shared resource group and are not yours to delete):

1. Navigate to **Resource groups** → `trn_<id>_rg`
2. Select the checkboxes next to `trn<id>loctest`, `trn<id>vmtest`, `trn<id>tagtest`, `trn<id>pip` (whichever succeeded/exist)
3. Click **Delete** at the top, type the resource group name to confirm, and click **Delete**

> Note: Exercises 1, 2, and 5 should have failed to deploy, so there may be nothing to clean up for those. Exercise 3's storage account did deploy successfully and does need deleting.

---

*Reference: Microsoft Learn — Azure Policy effects, Resource locks overview, Log Analytics query getting started.*
