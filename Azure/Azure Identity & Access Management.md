# Azure Managed Identities with Microsoft Entra

Authentication : Users and Groups 
Authorization : Roles

##### To implement IAM in Azure, there is service in azure called `Microsoft Entra ID`.
Previously  it was called Azure Active Directory. 

Users, Groups, Roles, Policies, 

`When one resource want to access another resource then that can be done via Service Principal & Managed Identities. `


## What is difference between Service Principal & Managed Identities?

**Whether we are using Service Principal or Managed Identities, underlying azure is gonna create a Service Principal.**

An Azure service principal is a secure, non-human digital identity created in Microsoft Entra ID. It allows applications, automated tools, and CI/CD pipelines to sign in and access or manage Azure resources programmatically without using personal user accounts.

If we create service principal then we need rotate the access keys as a DevOps Engineer.
But If we are creating Managed Identities then Azure takes care of this rotation. 


A **managed identity** is a special, automated type of [Microsoft Entra ID Service Principal](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops). The key difference is that Azure automatically manages a managed identity's credentials and lifecycle, whereas a standard service principal requires you to manually create, rotate, and secure its passwords, secrets, or certificates. 

Credential and Secret Management

- **Managed Identity:** No secrets to manage. Azure handles creation, rotation, and usage behind the scenes.

- **Service Principal:** You must manually generate and rotate client secrets or certificates before they expire to prevent application outages. [[1](https://www.atmosera.com/blog/azure-service-principal-vs-managed-identity/), [2](https://redriver.com/cloud/azure-managed-identity-vs-service-principal)]

Scope and Resource Association

- **Managed Identity:** Tied directly to Azure resources (either system-assigned to a single resource or user-assigned as a standalone resource). It is automatically deleted if the underlying system-assigned resource is deleted.

- **Service Principal:** Independent of any single Azure resource. It exists as a global application object in Microsoft Entra ID. [[1](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops), [2](https://redriver.com/cloud/azure-managed-identity-vs-service-principal), [3](https://www.atmosera.com/blog/azure-service-principal-vs-managed-identity/)]

Where They Can Be Used

- **Managed Identity:** Works exclusively for workloads and services running inside Azure.

- **Service Principal:** Can be used inside Azure, outside Azure, on local servers, or across different multi-cloud and CI/CD environments (like GitHub Actions or local scripts). [[1](https://redriver.com/cloud/azure-managed-identity-vs-service-principal), [2](https://www.oasis.security/blog/service-principal-vs-managed-identity-in-azure), [3](https://medium.com/@cloudwithaarya/service-principal-vs-managed-identity-in-azure-when-should-you-use-each-with-demo-0580bb22adc8)]



To connect a VM to Azure Blob to access a file in blob via managed identity:
1. Both needs to be present in resource group.
2. Go to VM Identity and `on` the toggle for System Assigned managed identity and save it.
3. Go to Storage Account and go to Access Control (IAM) and get to Add role assignment and assign the identity that we have created in previous step. (So simply we are assigning a role to the identity we have created)
4. Give role as owner (initially) & in Member's tab select Assign access to Managed Identity 
5. And In select members, in managed identity select VM and the VM that we have created & wanted to grant the blob storage.
6. 