Azure App Service is a platform that lets you run web applications, mobile back ends, and RESTful APIs without worrying about managing the underlying infrastructure. 
It's a fully PaaS solution.
Think of it as a powerful web hosting service that takes care of all the heavy lifting for you, so you can focus on creating great applications.

So basically the app service plan provides the compute. It abstracts layers like Storage/Network/Compute, Virtual Machine & Operating system.
We can simply choose our runtime environment like python, nodejs, java etc and run applications and their data on top of it.

Engineers can run even containerized application on app service.
![[Pasted image 20260818112331.png]]

Like VMs, App service also comes with different tier plans. These tiers represents the performance, features, size and price you pay.

We can run multiple applications on single app service plan.

We can opt for new service plan if we want to deploy our apps in different region or need underlying os to be changed.

Even if we are not running any applications, once the App service plan is deployed, we'll start paying for it. We cannot stop this one like we do for VM. So we need to choose plans wisely to optimize the cost.

# Azure App Service Plans

An **App Service Plan** is the server (or set of servers) you rent to run web apps.
All apps in the same plan share its CPU, memory and disk — you pay for the *plan*, not per app.

![[Pasted image 20260818113916.png]]
## Tier comparison

| Feature                          | Free (F1)          | Shared (D1) ⚠️ legacy | Basic (B1–B3) | Standard (S1–S3) | Premium (Pv3/Pv4)     | Isolated (Iv2 / ASE)           |
| -------------------------------- | ------------------ | --------------------- | ------------- | ---------------- | --------------------- | ------------------------------ |
| Machine type                     | Shared with others | Shared with others    | Dedicated     | Dedicated        | Dedicated, faster CPU | Your own private               |
| Daily CPU quota                  | 60 min/day         | 240 min/day           | None          | None             | None                  | None                           |
| Disk (approx.)                   | 1 GB               | 1 GB                  | 10 GB         | 50 GB            | 250 GB                | 1 TB                           |
| Custom domain                    | ❌                  | ✅                     | ✅             | ✅                | ✅                     | ✅                              |
| Free managed HTTPS cert          | ❌                  | ❌                     | ✅             | ✅                | ✅                     | ✅                              |
| Manual scale-out (max instances) | 1                  | 1                     | 3             | 10               | 30                    | 100                            |
| Autoscale                        | ❌                  | ❌                     | ❌             | ✅                | ✅                     | ✅                              |
| Deployment / staging slots       | 0                  | 0                     | 0             | 5                | 20                    | 20                             |
| Always On (app never sleeps)     | ❌                  | ❌                     | ✅             | ✅                | ✅                     | ✅                              |
| Automatic backups                | ❌                  | ❌                     | ❌             | Daily            | Many per day          | Many per day                   |
| Private VNet integration         | ❌                  | ❌                     | Limited       | Limited          | ✅                     | ✅ (full isolation)             |
| Relative cost                    | Free               | Cents                 | Low           | Medium           | High                  | Very high (fixed monthly base) |

## Which one to pick

| Your situation                                                    | Tier                        |
| ----------------------------------------------------------------- | --------------------------- |
| Learning, demo, throwaway                                         | **Free**                    |
| Dev/test, small internal tool                                     | **Basic**                   |
| Normal production site or API (zero-downtime deploys + autoscale) | **Standard** ← usual answer |
| High traffic, or must reach private databases/VNets               | **Premium v3/v4**           |
| Compliance requires full network isolation                        | **Isolated**                |

## Things that catch people out

- **Free tier's 60 CPU-minutes/day** is a hard stop — the app returns errors until the next day.
- **Slots only start at Standard.** If you want a staging URL you can swap into production, Basic won't do it.
- **Linux plans are cheaper** than Windows for the same tier.
- **You can change tier anytime** — scaling up/down takes a minute and needs no redeploy. So start small. Resizing a app plan will reboot our application.
- **Shared (D1) is being retired** — don't build anything new on it.
- Exact quotas and available SKUs (**Stock-Keeping Unit**) vary by region and change over time — confirm against the Azure pricing calculator / App Service limits docs before budgeting.

In App Service plan, scaling your application can be done in two ways:
1. Scale up (Vertical Scaling) : It is the process of changing your service plan to a higher tier that offers more CPU, RAM & Disk space. This is perfect for application that need more power, but not necessarily more instances.
2. Scale out (Horizantal scaling) : Adding more instances of your application & distributing the load across multiple servers. 

Azure App Service Plan allows us to configure autoscaling based on metrics like CPU usage or request queues, which can be crucial for handling sudden traffic spikes without manual intervention.


 ---

**Azure Container Instances** allows us to run containers directly without any orchestration layer. This is perfect for simple applications or batch jobs.

If we want simplified container orchestration, a solution that still abstracts some of the complexity, we could go for **Azure Container Apps**.
This service is suitable for microservices that requires scaling and orchestration but not the full capabilities of kubernetes.

Lastly for advanced container orchestration, azure offers **AKS**.

----

Creating App service : 
1. It is not necessary to create a app service plan prior to creating app service. We can create it on the go while creating the app service.
2. Name of App Service, Publish : Code (runtime environment) or Docker Container or Static Web App.


During the deployment of your web application, you need to perform testing in a production-like environment without affecting the live site. We will use **Deployment slots** for that.

---

### Securing App Service:
![[Pasted image 20260818131401.png]]

In Networking part of App Service we can disable the public access and add rules for accessing our deployed application on App Service with help of allow & deny.

---

## Custom Domain : 

Article : https://notes.kodekloud.com/docs/Updated-AZ-104-Microsoft-Azure-Administrator/Administer-PaaS-Compute-Options/Custom-Domains/page

The default domain granted by the App Service when we initially deployed app is azurewebsite.net domain.

We can bring our own domain to create branding. We need to validate domain before we add it in App Service. Custom Domain supports both A & CNAME mapping, but before that we need to add a TXT record to prove that we own the domain. Then we can add CNAME & A record for custom domain to our App Service.

Custom Domains are supported from basics plan onwards. So we can't use custom domains for free or shared tier.

----

## Backing up App Service:

App Service supports both manual and scheduled backup, which include the backup of the configuration, file contents and connected database.

Backup can be upto 10 Gb of app and database.
Full and partial backups can be configured.
We can restore the app to a previous restore point or create a web app altogether.

Backup is also plan dependent. It requires Standard or Premium plan.


---

## CI/CD :

We can integrate App Service with Github, or other VCS Providers.

CI/CD enhances code quality and accelerates delivery by automating integration and deployment processes. Azure App Service supports two primary deployment methods that help teams minimize downtime and improve reliability:

1. **Automated Deployment**  
    Developers push new code—including features, patches, and bug fixes—to repositories such as GitHub, Bitbucket, Local Git, or Azure Repos. When integrated with CI/CD, these updates are automatically propagated to the App Service with minimal impact on end users.
2. **Manual Deployment**  
    Developers store code in remote cloud storage services (e.g., OneDrive or Dropbox) or external Git repositories, then manually trigger updates to the App Service.

### Deployment Slots

Deployment slots enable you to create multiple environments (such as staging, QA, UAT, or development) within a single Azure App Service. Testing new code in a staging slot prior to a production swap minimizes downtime and avoids performance issues like cold starts. Key features of deployment slots include:

- **Environment Separation:**  
    Each slot represents an environment—production, QA, or development—allowing for isolated testing and validation.
- **Auto Swap:**  
    Automatically swap slots when validations are successful to eliminate service disruptions.
- **Rollback Capability:**  
    Quickly revert to a previous version by swapping back to the last known good configuration.
- **Service Plan Limitations:**
    - Free, Shared, and Basic: Deployment slots are not supported.
    - Standard: Supports up to 5 slots.
    - Premium and Isolated: Support up to 20 slots.

When creating a deployment slot, decide whether to clone the app configuration (from production or another slot) or start fresh. Understand which settings swap and which do not. Generally, swappable settings include general settings, WebJob contents, app settings, path mappings, hybrid connections, connection strings, service endpoints, handler mappings, and Azure CDN configurations. Non-swappable settings include publishing endpoints, scale settings, CORS, custom domains, IP restrictions, VNet integration, non-public certificates, always-on configuration, managed identities, TLS/SSL settings, diagnostic settings, and any setting ending with an extension version suffix.

![[Pasted image 20260818165354.png]]
Working with branches & slots :

You can configure separate deployment slots for different branches to facilitate parallel development. For example, create a “dev” branch in Azure Repos containing the same content as the main branch, and then add a corresponding deployment slot in your App Service.

---

