Helm is a package manager for k8s. 

Helm uses a packaging format called _charts_. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

Inside of this directory, Helm will expect a structure that matches this:

```
wordpress/  Chart.yaml          # A YAML file containing information about
the chart  LICENSE             # OPTIONAL: A plain text file containing the license for the chart  README.md           # OPTIONAL: A human-readable README file  values.yaml         # The default configuration values for this chart  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file  charts/             # A directory containing any charts upon which this chart depends.  crds/               # Custom Resource Definitions  templates/          # A directory of templates that, when combined with values,                      # will generate valid Kubernetes manifest files.  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

Helm Chart Hook : Even before our app gets up with helm , db can be backed up. So this is single chart hook usecase.

To create a initial basic skeleton for working with helm.
```
helm create /root/webapp-nginx
```

Each time we create a new release of same app, if our deployment name is nginx-deployment set in values.yml then we are not able to create new release as the deployment name is already been created earlier. 
For this we templatize the name of deployment.
```bash
name: {{ .Release.Name }}-nginx
```

To verify that the chart is correctly defined, we can also use this command:
```bash
helm lint ./nginx
```

Before actually installing this, we can do what is called a **dry run**. This **pretends** to install the package to the cluster, and it can catch things that Kubernetes would complain about in a real install.
```bash
helm install sre-1 ./student-api --dry-run
```


**Single Chart Problem:**
- Can't upgrade components independently
- Staging might need new API but old Vault

**Multiple Charts Solution:**
- Each component can have its own version matrix
- Canary deployments per component

```mermaid
flowchart TD
    Start(["helm upgrade"]) --> Load

    subgraph Load["1. Chart Loading & Validation"]
        L1["Load chart from local path or repository"]
        L2["Validate Chart.yaml, values.yaml, and templates"]
        L3["Merge values<br/>(--set, -f, --set-string, --set-file)"]
        L1 --> L2 --> L3
    end

    Load --> Render

    subgraph Render["2. Template Rendering"]
        R1["Render templates with merged values"]
        R2["Process helm.sh/hook annotations"]
        R3["Evaluate conditions and loops"]
        R4["Generate final Kubernetes manifest YAML"]
        R1 --> R2 --> R3 --> R4
    end

    Render --> Merge

    subgraph Merge["3. Three-Way Strategic Merge"]
        Old["Old manifest<br/>(currently deployed release)"]
        New["New manifest<br/>(new chart + values)"]
        Live["Live state<br/>(actual cluster resources)"]
        Compare["Determine required changes"]
        Old --> Compare
        New --> Compare
        Live --> Compare
    end

    Merge --> Diff

    subgraph Diff["4. Diff Generation"]
        Create["Create<br/>new resources"]
        Update["Update<br/>changed resources"]
        Delete["Delete<br/>removed resources"]
        Keep["Keep<br/>unchanged resources"]
    end

    Diff --> PreHook

    subgraph PreHook["5. Pre-Upgrade Hooks"]
        PH1["Run pre-upgrade hooks<br/>(Jobs / Pods)"]
        PH2{"Hooks succeed?"}
        PH1 --> PH2
    end

    PH2 -- "No + --atomic" --> Abort["Abort upgrade / rollback"]
    PH2 -- "Yes" --> Apply

    subgraph Apply["6. Ordered Resource Application"]
        A1["CRDs"]
        A2["Namespace / ServiceAccount"]
        A3["ConfigMap / Secret"]
        A4["PV / PVC"]
        A5["Deployment / StatefulSet / DaemonSet"]
        A6["Service / Ingress"]
        A7["NetworkPolicy"]
        A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> A7
    end

    Apply --> Rollout

    subgraph Rollout["7. Rolling Update Strategy"]
        D["Deployment:<br/>controlled pod replacement"]
        S["StatefulSet:<br/>ordered pod replacement"]
        DS["DaemonSet:<br/>node-by-node update"]
    end

    Rollout --> Wait

    subgraph Wait["8. Wait & Verification (--wait)"]
        W1["Wait for Pods to become Ready"]
        W2["Wait for rollout completion"]
        W3{"Resources healthy?"}
        W1 --> W2 --> W3
    end

    W3 -- "No + --atomic" --> Abort
    W3 -- "Yes" --> PostHook

    subgraph PostHook["9. Post-Upgrade Hooks"]
        PO1["Run post-upgrade hooks"]
        PO2["Clean up old hooks<br/>(hook-delete-policy)"]
        PO1 --> PO2
    end

    PostHook --> Store

    subgraph Store["10. Release Storage Update"]
        ST1["Save manifest in release storage<br/>(Secret or ConfigMap)"]
        ST2["Increment revision number"]
        ST3["Mark release as deployed"]
        ST1 --> ST2 --> ST3
    end

    Store --> Done(["Upgrade complete"])

    classDef critical fill:#fce4e4,stroke:#c62828,color:#5f0000;
    class Merge critical;
```