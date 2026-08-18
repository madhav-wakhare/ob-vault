Azure App Service is a platform that lets you run web applications, mobile back ends, and RESTful APIs without worrying about managing the underlying infrastructure. 
Think of it as a powerful web hosting service that takes care of all the heavy lifting for you, so you can focus on creating great applications.

So basically the app service plan provides the compute. It abstracts layers like Storage/Network/Compute, Virtual Machine & Operating system.
We can simply choose our runtime environment like python, nodejs, java etc and run applications and their data on top of it.

Like VMs, App service also comes with different tier plans. These tiers represents the performance, features, size and price you pay.




**Azure Container Instances** allows us to run containers directly without any orchestration layer. This is perfect for simple applications or batch jobs.

If we want simplified container orchestration, a solution that still abstracts some of the complexity, we could go for **Azure Container Apps**.
This service is suitable for microservices that requires scaling and orchestration but not the full capabilities of kubernetes.

Lastly for advanced container orchestration, azure offers **AKS**.

