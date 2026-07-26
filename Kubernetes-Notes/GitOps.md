Principles:
1. All Manifests Declarative in nature. 
2. Make use of Git. Desired state in stored in versioned control.
3. GitOps Operators also know as software agents, automatically pulls the desired state from Git & apply it in one or more environments or clusters. The GitOps Operator can run in one of the cluster and can make or push changes to other cluster as well.
4. The Final principle talks about reconciliation. The GitOps operator also make sure the entire systems is self-healing to reduce the risk of human error. The operator continuously loops through 3 steps : `observe, diff &  act`
   - In Observe step it check git repo for any changes in the desired state.
   - In Diff step it compares the resources received from the previous step to the actual state of the cluster.
   - And in act step, it uses a reconciliation function & tries to match the actual state to desired state.

Push-Based Deployment:
In a push-based deployment model, an external system—commonly part of a continuous delivery pipeline (GitHub Actions) —initiates the deployment process. For example, a successful commit to a Git repository or an earlier CI pipeline can trigger this process. This external system requires direct read-write access to the Kubernetes cluster, allowing it to push changes into the production environment.

`When using push-based deployment, you must expose cluster credentials outside of the cluster. Store these credentials securely within your CI/CD system to prevent unauthorized access.`

To secure push-based deployments, consider the following best practices:
- Encrypt credentials to shield them from exposure.
- Apply strict access controls to limit who can access these credentials.
- Regularly rotate credentials to minimize security risks.

Pull-Based Deployment:
In pull-based deployment, changes are applied from within the Kubernetes cluster. An operator running inside the cluster continuously monitors associated Git repositories and Docker registries for updates. When the operator detects a change, it synchronizes the cluster state accordingly.

`One significant advantage of this approach is enhanced security. With pull-based deployment, no external client holds administrative access to the cluster, thereby reducing the exposure of sensitive credentials and minimizing the attack surface.`


In a typical GitOps workflow, two Git repositories are employed:
- One repository holds the application code, resource definitions, configuration data, and container images.
- The other repository contains the Kubernetes manifests. These YAML files detail the desired state of the cluster, including key resources such as Deployments, Services, Ingresses, ConfigMaps, and Secrets.