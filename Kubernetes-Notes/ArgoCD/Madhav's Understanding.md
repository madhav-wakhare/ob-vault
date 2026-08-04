
- Argocd only uses helm template command under the hood to render the output instead of using any other helm commands.

- Revision stays **1**. No `sh.helm.release.v1.eso-config.v2` ever appears. If Argo CD were running `helm upgrade`, you'd get a new revision and a new secret on every sync. You won't. And in `managedFields`, the fields Argo CD wrote are attributed to `argocd-controller`, never to `helm`.

- The repo-server is stateless and has _no credentials for your target cluster_ — it physically cannot apply anything.


