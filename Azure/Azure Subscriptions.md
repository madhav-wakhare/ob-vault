
If we want to use any service & create resources in azure we can do it with help of Licenses:

Licenses types :
1. Subscriptions 
2. Per User (Applications)


### Azure Subscriptions : 

Subscription is like a billing account in azure. So Bills of services that we use are generated on basis of subscription. 

An Azure subscription is a logical container that links your identity to Microsoft's cloud services, serving as a boundary for billing, security, access control, and resource management. Every cloud resource, from virtual machines to databases, belongs to a single subscription.

#### Key Functions : 

- **Access Control:** Uses Role-Based Access Control (RBAC) to dictate who can view, create, or modify resources.
- **Scale and Quota:** Enforces technical limits and service quotas, helping organizations scale workloads safely.
- **Billing Boundary:** Tracks and aggregates costs, generating distinct invoices or usage reports per subscription.

Every subscription will have a subscription ID. The bills are always generated on subscription id.
Pay-As-You-Go is name of subscription here and it can be changed.

![[Pasted image 20260822152413.png]]![[Pasted image 20260822151705.png]]
If we don't want subscription we can cancel it anytime with help of Cancel subscription by clearing off pending bills.

There are 3 types of Subscriptions :
1. Free trail : Personal level
2. Pay-as-you-go : Monthly bills based on usage , Pay as you go account can be cancelled after settling bills.
3. Enterprises : Lock in Period for certain period of time (years). Azure heavy discounts can be granted.  We can't cancel the subscriptions at any time.

Any resources will be created under a subscription only so that bills can be generated.

---

Any cloud services are of 3 types :
1. IaaS
2. PaaS
3. Saas

Licenses can be purchased on 2 types :
1. Per user license : company purchased 100 licenses of Microsoft 365 per user & there will be **fixed fee** per month for the same. (Mostly SaaS comes in this category)
2. Cloud Based Resource Consumption : Monthly bills based on consumptions. (IaaS & PaaS)








