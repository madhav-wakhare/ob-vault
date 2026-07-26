# Helm Helper Files and Named Templates

#kubernetes #helm #devops #templating

> [!info] Summary
> Helm lets you define reusable template snippets, similar to functions, in a file called `_helpers.tpl`. Instead of repeating the same YAML in multiple places, you define it once and call it wherever needed. This follows the DRY principle — Don't Repeat Yourself.

---

## The Problem: Repetitive Labels

Kubernetes objects use labels — key-value pairs — for organizing and selecting resources. In a typical `deployment.yaml`, the same labels often need to appear in multiple places:

- `metadata.labels`
- `spec.selector.matchLabels`
- `spec.template.metadata.labels`

```yaml
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
```

> [!warning] Why this is risky
> As charts grow, it's easy to forget which value was used where — for example, accidentally using `.Chart.Name` instead of `.Release.Name` for the instance label. Repeated code means repeated chances for mistakes.

---

## The Solution: Helper Files

> [!note] Key Convention
> Helm ignores any file in `templates/` that starts with an underscore. This makes `_helpers.tpl` the standard place to store shared template logic.

What goes in this file is comparable to functions in regular programming — write the logic once, call it from anywhere.

### Defining a Named Template

```yaml
{{/*
Here, we generate selector labels. It's recommended to include comments
so others know what this section does.
*/}}
{{- define "nginx.labels" }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

- `{{- define "nginx.labels" }} ... {{- end }}` creates a named template called `nginx.labels`.
- This is a reusable block you can call from any other template file.

> [!tip] Naming Convention
> Prefix the template name with the chart's name — `chartname.templatename`.
>
> If a chart depends on sub-charts, and those sub-charts also define a template simply called `labels`, they could override each other and cause confusing bugs. Using `nginx.labels` instead of `labels` keeps names unique across the whole chart and sub-chart tree.

---

## Calling a Named Template

Use the `include` keyword to call a template from any other file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
  labels:
    {{- include "nginx.labels" . | indent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "nginx.labels" . | indent 6 }}
  template:
    metadata:
      labels:
        {{- include "nginx.labels" . | indent 8 }}
```

### Syntax Breakdown

| Piece | Purpose |
|---|---|
| `include "nginx.labels"` | Calls the named template by name |
| `.` (the dot) | Passes the top-level scope into the template |
| `\| indent 4` | Pipes the output through the `indent` function to add spacing |
| `{{-` (dash) | Strips extra blank lines from the output |

> [!question] Why is the dot needed?
> The named template uses `.Chart.Name` and `.Release.Name`, but it needs to know where to pull those values from.
>
> Passing `.` gives it the top-level scope, so Helm knows to fetch `.Chart.Name` from the current top-level chart, not from some unrelated context.
>
> If a named template is called from a child chart, without scope Helm would not know whether to use the parent chart's name or the child chart's name. Passing `.` resolves that ambiguity.

---

## Reusing Across Multiple Files

The real value shows when the same named template is reused across different manifest files, not just within one.

### Example: service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-nginx
  labels:
    {{- include "nginx.labels" . | indent 4 }}
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "nginx.labels" . | indent 4 }}
```

Same `nginx.labels` template, reused again, with no duplication.

---

## Result

Running:
```bash
helm template ./nginx
```

Generates the full expanded manifest with labels filled in automatically, even though the source files only contain a handful of `include` lines instead of repeated blocks.

> [!success] Benefit Summary
> - Fewer lines written than lines generated, consistently, every time.
> - No need to remember which field was used last time.
> - Update the label logic once in `_helpers.tpl`, and it propagates everywhere it's used.

---

## Quick Reference

| Concept | Description |
|---|---|
| Helper File | A file, commonly `_helpers.tpl`, storing reusable template logic. Ignored by Helm's renderer since it starts with an underscore. |
| Named Template | A block defined via `{{- define "name" }} ... {{- end }}` — comparable to a function. |
| `include` | Keyword used to call a named template from another file. |
| Scope (`.`) | Passed into the template so it knows which context — `.Release`, `.Chart`, etc. — to pull data from. |
| Naming Convention | `chartname.templatename` — avoids naming collisions with sub-charts. |

---

## How to Decide What Goes in Helper Files

Ask: what will be reused repeatedly across template files? Another approach is reviewing charts written by experienced Helm developers to see what they typically extract into helpers.

---

## Related Notes
- [[Helm Chart Structure]]
- [[Kubernetes Labels and Selectors]]