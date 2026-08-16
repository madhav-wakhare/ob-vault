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



### Connecting a VM to Azure Blob Storage via Managed Identity

**Goal:** Allow a VM to access a file in Azure Blob Storage without using storage account keys or credentials — using Azure AD (Managed Identity) instead.

#### Prerequisites

- Both the **VM** and the **Storage Account** should ideally be in the same **Resource Group** (not mandatory, but keeps things organized and easier to manage).

---

#### Step 1: Enable Managed Identity on the VM

- Go to your **VM** → **Identity** (under Settings).
- Under **System assigned**, toggle **Status** to **On**.
- Click **Save**.
- This creates an identity _for the VM itself_ in Azure AD — no separate credentials needed. Azure handles the identity lifecycle (created/deleted with the VM).

#### Step 2: Assign Role to the Identity on the Storage Account

- Go to your **Storage Account** → **Access Control (IAM)**.
- Click **Add** → **Add role assignment**.
- This is where we tell Azure: _"this identity is allowed to do X on this storage account."_

#### Step 3: Choose the Role

- In the **Role** tab, select an appropriate role.
    - You mentioned **Owner** — works for testing, but it's overly permissive (grants full control, not just data access).
    - **Better practice:** use **Storage Blob Data Reader** (read-only) or **Storage Blob Data Contributor** (read/write) — these are scoped specifically to blob data access.
- Click **Next** to go to the **Members** tab.

#### Step 4: Assign Access to Managed Identity

- In **Members**, select **Assign access to** → **Managed Identity**.
- Click **+ Select members**.

#### Step 5: Select the VM's Identity

- In the **Managed identity** dropdown, filter by **Virtual Machine**.
- Select the specific VM you enabled the identity for in Step 1.
- Click **Select** → **Review + assign**.

---

#### 💡 Notes / Best Practices

- **Avoid Owner role** for production — it grants management-plane access (delete storage account, change access policies, etc.), not just data access.
- Prefer:
    - `Storage Blob Data Reader` — read-only access to blobs
    - `Storage Blob Data Contributor` — read/write access to blobs
- Once role is assigned, your VM can use the **Azure Instance Metadata Service (IMDS)** endpoint to fetch a token and authenticate to Blob Storage — no keys, no secrets stored on the VM.
- Role assignments can take a few minutes to propagate.