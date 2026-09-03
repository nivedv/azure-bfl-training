# Participant Reference Handout
## Session 2 — Azure Networking, Identity & Security
### BFL Azure Cloud & Microsoft AI Foundry Training Program

**Author:** Nived Varma

> This handout is your take-home reference for Session 2. It summarizes the six topics covered, gives you quick-reference tables, and answers the questions participants most commonly ask. All facts below have been cross-checked against official Microsoft Learn documentation as of September 2026.

---

## 1. Azure Virtual Networks & Subnets

**Quick reference**

| Concept | Key Fact |
|---|---|
| Virtual Network (VNet) | An isolated, private network boundary in Azure, scoped to one region and one subscription. Address space is defined in CIDR notation. |
| Subnet | A range carved out of the VNet's address space; the unit you attach NSGs, route tables, and resources to. |
| Reserved addresses per subnet | **5**, regardless of subnet size — the first four addresses plus the last address. |
| Reserved address breakdown (example: `192.168.1.0/24`) | `192.168.1.0` — network address · `192.168.1.1` — default gateway · `192.168.1.2` & `192.168.1.3` — mapped to Azure DNS · `192.168.1.255` — broadcast address. |
| Default connectivity | All resources within one VNet can reach each other by default — segmentation (subnets + NSGs) is something you add deliberately. |

**Session scenario recap:** `web-subnet` (10.20.1.0/24) and `data-subnet` (10.20.2.0/24), each governed by its own NSG, enforcing that the database tier is never directly reachable from the internet.

---

## 2. DNS & VNet Connectivity

**Quick reference**

| Concept | Key Fact |
|---|---|
| Azure-provided DNS | Automatic, works within a single VNet, Azure-generated hostnames — doesn't cross VNet boundaries. |
| Azure Private DNS Zone | Your own custom internal domain (e.g. `bfl.internal`), resolvable across every VNet you explicitly link to it. |
| Azure Public DNS Zone | Hosts publicly resolvable domains (e.g. `bfl.com`) — a separate use case from internal name resolution. |
| VNet Peering | Connects two VNets over Microsoft's private backbone (not the public internet) using private IP addresses — no VPN gateway required. |
| Peering transitivity | **Non-transitive.** If VNet A peers to VNet B, and B peers to C, A cannot reach C automatically. A direct A–C peering, or routing through a firewall/NVA in a hub, is required. |
| Peering creation | Must be created on **both** VNets. The Azure Portal wizard does this in one action; CLI/PowerShell requires a separate command on each VNet. |

**Session scenario recap:** `trainer_bfl-hub-vnet` peered to `trainer_bfl-spoke-vnet`, plus a Private DNS Zone (`bfl.internal`) linked to the spoke for internal name resolution.

---

## 3. Azure Load Balancing

**Quick reference**

| Concept | Key Fact |
|---|---|
| Public Load Balancer | Frontend has a public IP — for internet-facing tiers. |
| Internal Load Balancer | Frontend IP lives inside a VNet subnet, no public IP — for backend/internal services. |
| Standard SKU | Production default. 99.99% SLA, supports Availability Zones, **secure by default** — closed to inbound traffic unless an NSG explicitly allows it. |
| Basic SKU | **Retired as of 30 September 2025.** New Basic Load Balancers can no longer be deployed, and existing ones can no longer be used past that date (Azure Cloud Services extended-support deployments were exempted). All new designs must use Standard SKU. |
| Core components | Frontend IP → Backend Pool → Health Probe → Load Balancing Rule. |
| Health probe behavior | An unhealthy backend instance is automatically removed from rotation within seconds of a failed probe. |

**Session scenario recap:** `trainer-lb-web` (Standard SKU, public) in front of the web tier; the hands-on lab used an **internal** Standard Load Balancer for the claims microservice.

---

## 4. Microsoft Entra ID — Authentication & Authorization

**Quick reference**

| Concept | Key Fact |
|---|---|
| Microsoft Entra ID | Microsoft's cloud identity platform (formerly Azure AD) — the directory behind sign-ins to Azure, Microsoft 365, and any registered application. |
| Tenant | An organization's dedicated, isolated instance of Entra ID. |
| Authentication | "Who are you?" — sign-in, MFA, Conditional Access. Produces a token proving identity — nothing about permissions. |
| Authorization | "What are you allowed to do?" — in Azure, this is Azure RBAC, applied after authentication succeeds. |
| App Registration | An application's identity record in Entra ID — distinct from a role assignment, which governs what that identity (or a user) can do. |

**Common failure pattern:** a user signs in successfully (including MFA) but sees "access denied" opening a resource — this is almost always an **authorization** gap (missing RBAC role at the relevant scope), not an authentication failure.

---

## 5. RBAC & Managed Identity

**Quick reference**

| Concept | Key Fact |
|---|---|
| Role assignment | Consists of exactly **three elements**: a **security principal** (user, group, service principal, or managed identity), a **role definition** (the permissions), and a **scope** (where it applies). |
| Scope hierarchy | Management Group → Subscription → Resource Group → Resource. Permissions granted higher up flow down. |
| Best practice | Assign at the narrowest scope that satisfies the need. |
| Managed Identity | An automatically managed Entra ID identity for an Azure resource — no credential a developer stores, sees, or rotates. Available at no extra cost. |
| System-assigned identity | Tied 1:1 to the resource's lifecycle; created and deleted with it; usable only by that resource. |
| User-assigned identity | A standalone identity resource that can be assigned to multiple resources; managed independently. |
| How it works | The resource requests a token from Microsoft Entra ID (via the Instance Metadata Service); the token is presented to the target (e.g. Key Vault) — no password ever exists in the flow. |

**Session scenario recap:** `trainer-vm-web`'s system-assigned identity was granted the **Key Vault Secrets User** RBAC role, scoped to `trainer-kv-bfl-web` only.

---

## 6. Defense-in-Depth & Microsoft Defender for Cloud

**Quick reference**

| Concept | Key Fact |
|---|---|
| Defense-in-depth | Five layers working together — Identity, Network, Compute, Data, Monitoring. No single layer is the whole defense. |
| Microsoft Defender for Cloud | A cloud-native application protection platform (CNAPP) combining CSPM (posture management), DevSecOps, and CWPP (workload protection). |
| Foundational CSPM | **Free.** Provides Secure Score, recommendations, asset inventory, and regulatory compliance mapping. |
| Defender plans (Servers, Key Vault, Storage, SQL, etc.) | **Paid**, billed per resource/hour — add active threat protection for that specific resource type. |
| Secure Score | A percentage reflecting overall posture; improves as recommendations are remediated. Refresh is **not instant** — allow several hours after a change. |
| Upcoming change to note | From **27 October 2026**, Foundational CSPM moves to an **opt-in** model for *new* Azure subscriptions (it will no longer be on by default). It remains free — you simply need to turn it on. Existing subscriptions that already have it enabled are unaffected. |

---

## Frequently Asked Questions

**Q1. Why segment the network into subnets instead of keeping everything in one flat VNet?**
Segmentation lets each tier carry its own NSG rules, so a compromise in one tier (e.g. the web tier) doesn't automatically grant network-level access to another (e.g. the database tier). It's enforcement at the network layer, independent of what the application code does.

**Q2. Does Azure really reserve 5 IP addresses in every subnet, even a small one?**
Yes — this is fixed by Azure and applies to every subnet regardless of size, down to the smallest supported subnet. Plan your address space with this overhead in mind (a /24 gives 251 usable addresses, not 256).

**Q3. If Hub is peered to Spoke-A and Spoke-B, can Spoke-A reach Spoke-B?**
No. VNet Peering is non-transitive. Spoke-A and Spoke-B would need either a direct peering between them, or traffic routed through a firewall/NVA sitting in the hub that has reachability to both.

**Q4. What's the difference between Azure Private DNS Zones and Azure Public DNS Zones?**
Private DNS Zones resolve internal names (e.g. `db.bfl.internal`) across VNets you link to the zone — purely for internal use. Public DNS Zones host domains resolvable from the internet (e.g. `bfl.com`). They solve different problems and are configured separately.

**Q5. Is Basic Load Balancer still usable?**
No — Basic Load Balancer was fully retired on 30 September 2025. It can no longer be deployed or used (except for pre-existing Azure Cloud Services extended-support workloads). All new designs should use Standard SKU.

**Q6. Why does Standard Load Balancer need an NSG to pass any traffic at all?**
Standard SKU is secure-by-default — inbound traffic to a Standard Load Balancer or Standard public IP is blocked unless an NSG explicitly allows it. This is a deliberate zero-trust design choice, unlike Basic SKU, which was open by default.

**Q7. Is Authentication the same as Authorization?**
No. Authentication proves identity (who you are — sign-in, MFA). Authorization decides what that identity is allowed to do (Azure RBAC). A user can authenticate successfully and still be denied access if no suitable role is assigned.

**Q8. What exactly are the three parts of an RBAC role assignment?**
Security principal (who/what is receiving access), role definition (what permissions), and scope (where those permissions apply). All three are required — Microsoft Learn's official phrasing is: "a role assignment consists of three elements: security principal, role definition, and scope."

**Q9. What's the real difference between system-assigned and user-assigned managed identity?**
System-assigned is created with the resource and deleted when the resource is deleted — it can only be used by that one resource. User-assigned is a standalone Azure resource with its own lifecycle, and can be attached to multiple resources at once — useful when several VMs need identical permissions.

**Q10. Does using Managed Identity cost anything extra?**
No — managed identities are available at no additional cost, on top of whatever resource (VM, App Service, etc.) they're attached to.

**Q11. If I enable a system-assigned managed identity on a VM but access still fails, what's usually wrong?**
Almost always a missing or incorrectly scoped role assignment — the identity exists, but it hasn't been granted a role (e.g. "Key Vault Secrets User") on the target resource. Role assignment propagation can also take a few minutes.

**Q12. Is Microsoft Defender for Cloud free?**
Foundational CSPM (Secure Score, recommendations, asset inventory) is free for every subscription. Turning on a specific Defender plan for a resource type (Servers, Storage, Key Vault, SQL, etc.) adds paid, per-resource active threat protection.

**Q13. Why didn't the Secure Score change during the demo?**
Secure Score and recommendation refresh is not instantaneous — it can take several hours to reflect a configuration change. Don't expect a live score movement inside a training session; the recommendations list itself is the more reliable thing to demo live.

**Q14. What does "defense-in-depth" actually mean in practice, beyond a buzzword?**
It means no single control — not NSGs, not RBAC, not Managed Identity, not Defender for Cloud alone — is treated as sufficient on its own. Identity, network, compute, data, and monitoring layers are designed to back each other up, so that a failure in one layer doesn't equal a full compromise.

---

*Reference: Microsoft Learn — Azure Virtual Network subnet documentation, Azure Virtual Network FAQ (peering transitivity), Azure Load Balancer overview and SKU documentation, Azure Basic Load Balancer retirement announcement, Microsoft Entra ID overview, Azure RBAC overview ("How Azure RBAC works"), Managed identities for Azure resources overview, Microsoft Defender for Cloud get-started and Foundational CSPM documentation (learn.microsoft.com). Verified September 2026.*
