# Resource move policy issue
This section investigates options to enforce policies in situation of resource move between unmanaged and managed subscriptions.

## Problem statement
Consider managed subscription with Azure Policies in place defined by central team to enforce limitations on VNETs, usage of public IPs, enforced auditing and encryption settings. Also consider there are unmanaged subscriptions in the same tenant with no policies.

User can be contributor in both unmanaged and managed subscription and initiate movement of resources between subscriptions. During this proces Azure Policies in destination subscription are not checked/enforced creating security risks. One example is data exfiltration - managed subscription might be locked from accessing Internet without going throw NGFW/DLP platform. User can move VM with public Internet access (unmanaged VNET) to managed subscription, take disk from managed corporate VM, attach it to this unmanaged resource and copy data from disk to some endpoint in Internet.

## Potential solutions and workarounds

Custom RBAC - proactive, but not complete, [details here](#custom-rbac) 
- Can prevent movement of unwanted resources by removing permisisons to write such objects (eg. VNET, Public IP)
- Does not prevent writing resources that must be allowed, but some properties enforced (eg. enforced TLS on storage account, tag structure, audit logs enabled)
- Require change on existing subscriptions by moving from Contributor role to custom role

Ideas:
- modify resource for policy to kick in
- rewrite key policies to simple code and:
  - listen on change feed and generate alert using Functions or/and Logic Apps
  - use scheduled jobs (Funcitons, Logic App or Automation script)
- can change feed or activity log recognize move operation?

### Custom RBAC
TBD