
### Resource Group

A container which holds resources. Any resources created in azure is a part of a Resource group.

A **Resource Group** in Azure is a logical container for resources that share the same lifecycle, permissions, and policies. It helps you organize and manage related Azure resources efficiently. Resources within a group can be deployed, updated, and deleted together as a single management unit.

### Key Points about Resource Groups:

- **Lifecycle Management:** Resources within a group can be managed collectively, making it easy to handle deployments, updates, and deletions.
- **Resource Organization:** Grouping resources based on projects, environments, or applications helps keep your Azure environment well-organized.
- **Role-Based Access Control (RBAC):** Permissions and access control are applied at the resource group level, allowing you to manage who can access and modify resources within a group.

Resource & Resource Group share 1:1 mapping. If any resource is a part of RG then it can't be part of other RG.

Ex : If I want a Dev Resource Group and have separate teams under that A, B & C.
if there are dedicated 10 VMs & want to give permission to only team A then I can do it in single go with help of Resource Groups policies.


