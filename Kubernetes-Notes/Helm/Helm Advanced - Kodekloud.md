Whenever we do helm upgrade, a database can be automatically backed up before that happens with help of `hooks`.
So we have a way to restore data from backup incase something goes wrong.

When we install a chart (eg: nginx) with command :
`helm install chart-1 ./nginx`
Then the problem with this is the deployments in this nginx chart name is hardcoded so the helm throws an error as deployment with name <name-deployment> already exists.
To resolve this we templatize the name of deployment:

`name : {{ .Release.Name }}-nginx` -> Becomes `chart-1-nginx`

This will create deployment name based on release name so that the issue with same deployment name will not occur.
We should also do this with all resources we are creating with helm.

`Helm doesn't allow us to install two releases with same name.`

{{ }} -> Template Directive (Go Templating language).
Release.Name -> Object between the curly braces that we would like to access.
. -> Refers to root level or top level scope.


Anything that defines with Values refers to properties that is defined in values.yml.

Writing Dictionary in vaules.yml is recommended that repeating the object. (ex : .Vaules.image.repo)


When we install a helm chart, helm uses the files in template directory and then combines it with the information about the release (such as release name) & the data stored in values.yml file & chart details (if any). 


helm lint, helm template, helm dry run (commands to verify if our charts are correct syntactically and logically)

helm template -> to render the template of entire resources in helm chart. Replaces placeholders with actual values in vaules.yml

helm dry run -> to catch k8s issues for our helm chart.

We can also append the --debug flag in front of the helm template command to troubleshoot about issues and know more about exact issue.


Functions:
If we don't provide a value for container image it should fallback to some value, function helps us in that.

We can add a function called `default` before the variable & specify a default value for it.
Note that the value should always be in quotes "nginx", so anything not in quotes is considered as variable.
```yml
image: {{ .Values.image.repository | default "nginx" }}
```

In Helm, the `include` function imports a named template and returns its output as a string. This allows developers to pass the imported block through template pipelines and functions, such as `indent` or `nindent`, which is crucial for maintaining valid Kubernetes YAML formatting.

```yml
{{ include "TEMPLATE_NAME" CONTEXT }}
```

- **`   TEMPLATE_NAME`**: The exact string name of the template defined elsewhere (usually in `_helpers.tpl`).


- **`CONTEXT`**: The scope or data passed to the template (typically standard dot `.` to pass the current global context

Named template in _helper.tpl:
```yml
{{- define "mychart.appMetadata" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

templates/deployment.yaml:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    {{- include "mychart.appMetadata" . | nindent 4 }}
```

Pipelines:
Similar to using pipe in linux for forwarding one command output to another command.
So with pipelines, we can now pipe multiple functions one after the other.
```yml
image: {{ .Values.image.repository | default "nginx" | upper }}
```

Conditionals:
![[Pasted image 20260726224707.png]]

Only render the label if it is present in vaules.yml.
The if block is also within the curly braces.

A Dash which matter: 
If & end block leaves lines empty in final output after rendering, so we add a dash {{- if .Values.label }} & {{- end }} right after the curly braces to indicate helm to trim those out when the files are generated.


With Block:
Setting scope by `with block` so that traversing everytime to pick values from values.yml of nested objects. `$` means Root here.
![[Pasted image 20260726231035.png]]


Range :
Iterate through multiple values like for block in programming.
![[Pasted image 20260726231814.png]]


### Named Templates:
![[Pasted image 20260726235359.png]]

`.` in this service.yaml is to transfer the Release name context to helper file.

![[Pasted image 20260727000111.png]]

	Indentation with help of `include` instead of `template`.


### Hooks :
![[Pasted image 20260727001845.png]]