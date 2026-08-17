![[Pasted image 20260815190712.png]]

**Tenant**: A tenant is a dedicated and isolated instance of the Azure Active Directory (Azure AD)/Microsoft Entra ID service that an organization receives when it signs up for a Microsoft cloud service such as Azure, Microsoft 365, or Dynamics 365. Each tenant has its own identity and access management scope, and is distinct and separate from other tenants. A tenant is also associated with a **unique tenant ID**, which is a globally unique identifier (GUID) that identifies the tenant in Azure AD.

**Directory**: A directory is a container for objects such as users, groups, and applications, and is used to manage access to resources in Azure. A directory is also associated with a unique directory ID, which is a GUID that identifies the directory in Azure AD.

**Domain**: A domain is a name that identifies a group of resources on the internet, such as a website or an email server. In Azure, a domain is used to verify ownership of a custom domain name and to create user names and email addresses for users in a tenant. A domain can be associated with one or more tenants, and each tenant can have multiple domains.

The main difference between a tenant and a directory is that a tenant is a dedicated and isolated instance of Azure AD/Entra ID, while a directory is a container for objects such as users, groups, and applications. A tenant can contain one or more directories, and each directory can contain one or more tenants.

In summary, a tenant is a dedicated and isolated instance of Azure AD that an organization receives when it signs up for a Microsoft cloud service, while a directory is a container for objects such as users, groups, and applications, and is used to manage access to resources in Azure. A domain is a name that identifies a group of resources on the internet, and is used to verify ownership of a custom domain name and to create user names and email addresses for users in a tenant.