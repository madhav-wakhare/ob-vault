
ARM is responsible for creating resources on Microsoft Azure.

Users can create azure resource by any of the following :
1. UI/Portal
2. Azure CLI
3. ARM Template
4. Bicep
5. SDK
6. Terraform / Iac platform 

All of these services talk to Azure Resource Manager.

User requests resource by any of the means -> ARM -> Azure Resource created.

- Why do ARM exists when we can create resources simply by any method ?
  Standardization. 

- What are ARM Templates?
  JSON Templating standard which are able to create resources on Microsoft azure.

![[Pasted image 20260816152242.png]]

We can override the values in ARM Templates with help of parameters in order to make it reusable with different set of values.
We can have variables and functions in the ARM Templates as well.



