---
title: "From ecsctl to Helm — Pahud's bot-clone pattern, ported to GCP/k8s"
description: "The template + --set override pattern Pahud uses to spawn dozens of Discord bot clones on AWS ECS Fargate with oablab/ecsctl maps almost line-for-line to Helm on GCP/GKE. Working chart included. AWS Secrets Manager JSON-field selector → External Secrets Operator `property:`. Cost comparison."
---

# From ecsctl to Helm — the clone-via-template pattern, ported to GCP/k8s

In a recent screenshot from Pahud Hsieh's Telegram community, he ran a single
command to spawn a new Discord bot persona on AWS ECS Fargate using
[oablab/ecsctl](https://github.com/oablab/ecsctl):

```bash
ecsctl apply -f chaodu.yaml \
  --set metadata.name=openab-chaodu2 \
  --set spec.secrets.DISCORD_BOT_TOKEN='arn:aws:secretsmanager:...:CHAODU2_BOT_TOKEN::'
```

One template, one `--set` override, a new bot. Pahud's group cheered — *"我也要幾個分身就有幾個分身"* ("I can have as many clones as I want").

What people noticed less is that this pattern — **template + override = N clones** — is exactly what **Helm has been doing on Kubernetes since 2016**. ecsctl brings it to ECS for the first time; Helm has been doing it for almost a decade on GCP/AWS/Azure k8s.

This document maps the AWS pattern to its GCP/Helm equivalent line by line, includes a complete working chart, and reports the cost difference.

## The pattern, decomposed

What Pahud's `ecsctl apply --set` actually does:

1. Reads a YAML **template** (`chaodu.yaml`)
2. Applies CLI **overrides** (`--set k=v` patches the parsed YAML)
3. Renders the final task definition
4. Calls the cloud API to provision the resource

This is identical to Helm's flow:

1. Reads a **chart** (a directory of YAML templates + a `values.yaml`)
2. Applies CLI **overrides** (`--set k=v` patches values.yaml)
3. Renders final Kubernetes manifests through Go templates
4. Calls the Kubernetes API to provision the resources

The mental model is the same. Only the runtime differs.

## Field-by-field correspondence

| Pahud's ecsctl | Helm equivalent |
|---|---|
| `ecsctl apply -f chaodu.yaml --set metadata.name=openab-chaodu2` | `helm install openab-chaodu2 ./openab-bot --set name=openab-chaodu2` |
| `--set spec.secrets.DISCORD_BOT_TOKEN=arn:...:CHAODU2_BOT_TOKEN::` | `--set secrets.discordToken=chaodu2-bot-token` (via External Secrets Operator) |
| `ecsctl get openab-chaodu2` | `helm status openab-chaodu2` plus `kubectl get pods -l app=openab-chaodu2` |
| `ecsctl log openab-chaodu2 -f` | `kubectl logs -f -l app=openab-chaodu2` |
| `ecsctl exec openab-chaodu2 bash` | `kubectl exec -it deployment/openab-chaodu2 -- bash` |
| `ecsctl cp file.txt openab-chaodu2:/tmp/` | `kubectl cp file.txt openab-chaodu2-xxxxx:/tmp/` |
| `ecsctl delete openab-chaodu2` | `helm uninstall openab-chaodu2` |
| Alias system (cluster/service/container) | namespace + label selector + release name (three independent mechanisms) |

Every command has a near-identical Helm answer, often older and more battle-tested.

## What about the Secrets Manager JSON-field trick?

Pahud stores N bot tokens as named fields inside a single AWS Secrets Manager
JSON object (`oab-dHVVk4`), then references each field with the
`arn:...:secret:NAME:KEY::` syntax. The motivation is cost: AWS Secrets Manager
charges $0.40 per secret per month, so consolidating 10 tokens into one secret
saves $3.60/month.

GCP Secret Manager has no equivalent of the `:KEY::` selector — but it doesn't
need one. GCP charges $0.06 per active secret version (with 6 free), so
**N separate secrets is the natural and cheap choice on GCP**.

If you genuinely need the "one secret, many keys" model on GCP (for example,
migrating an AWS workload without restructuring), the External Secrets Operator
exposes a `property:` field that selects a JSON key from a single Secret Manager
secret:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: chaodu2-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-store
    kind: SecretStore
  target:
    name: chaodu2-secrets
  data:
    - secretKey: DISCORD_BOT_TOKEN
      remoteRef:
        key: oab-tokens                   # one Secret Manager secret
        property: CHAODU2_BOT_TOKEN        # ← AWS-style JSON-key selector
```

So the JSON-field pattern *is* portable; it just lives in the k8s CRD layer
rather than the cloud provider's native API.

## Full working Helm chart

The chart below replicates Pahud's `chaodu.yaml` template. Drop it in a
`./openab-bot/` directory and `helm install` against any k8s cluster
(GKE Autopilot, EKS, kind, k3s).

### `openab-bot/Chart.yaml`

```yaml
apiVersion: v2
name: openab-bot
description: One Discord bot persona of the OpenAB fleet
type: application
version: 0.1.0
appVersion: "beta"
```

### `openab-bot/values.yaml`

```yaml
# Default values. Override per-clone with --set or -f overrides.yaml.

name: chaodu                              # k8s resource name suffix
image:
  repository: ghcr.io/openabdev/openab
  tag: beta
  pullPolicy: IfNotPresent

resources:
  cpu: "2"                                # 2 vCPU, matches Pahud's 2048
  memory: 4Gi                             # matches Pahud's 4096

# OpenAB runtime config (becomes plain env vars)
config:
  backendAgent: openab
  agentName: chaodu                       # the base persona name
  ghpoolUrl: http://ghpool.openab.local:8080
  stateBucket: openab-state-pahud

# Secret references (each clone overrides discordTokenKey)
secrets:
  storeName: gcp-store                    # External Secrets Operator SecretStore
  bundleSecret: oab-tokens                # the bundled JSON secret in GCP SM
  discordTokenKey: CHAODU_BOT_TOKEN       # which JSON field to pick
  sttKey: STT_API_KEY

serviceAccount:
  create: true
  name: ""                                # defaults to fullname
  annotations:
    iam.gke.io/gcp-service-account: openab-bot@PROJECT.iam.gserviceaccount.com
```

### `openab-bot/templates/_helpers.tpl`

```yaml
{{- define "openab-bot.fullname" -}}
{{- printf "openab-%s" .Values.name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "openab-bot.labels" -}}
app.kubernetes.io/name: openab-bot
app.kubernetes.io/instance: {{ include "openab-bot.fullname" . }}
app.kubernetes.io/managed-by: helm
app: {{ include "openab-bot.fullname" . }}
{{- end -}}
```

### `openab-bot/templates/external-secret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "openab-bot.fullname" . }}-secrets
  labels:
    {{- include "openab-bot.labels" . | nindent 4 }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: {{ .Values.secrets.storeName }}
    kind: SecretStore
  target:
    name: {{ include "openab-bot.fullname" . }}-secrets
  data:
    - secretKey: DISCORD_BOT_TOKEN
      remoteRef:
        key: {{ .Values.secrets.bundleSecret }}
        property: {{ .Values.secrets.discordTokenKey }}
    - secretKey: STT_API_KEY
      remoteRef:
        key: {{ .Values.secrets.bundleSecret }}
        property: {{ .Values.secrets.sttKey }}
```

### `openab-bot/templates/serviceaccount.yaml`

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ default (include "openab-bot.fullname" .) .Values.serviceAccount.name }}
  labels:
    {{- include "openab-bot.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}
{{- end -}}
```

### `openab-bot/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "openab-bot.fullname" . }}
  labels:
    {{- include "openab-bot.labels" . | nindent 4 }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ include "openab-bot.fullname" . }}
  template:
    metadata:
      labels:
        {{- include "openab-bot.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ default (include "openab-bot.fullname" .) .Values.serviceAccount.name }}
      containers:
        - name: openab
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: OPENAB_BACKEND_AGENT
              value: {{ .Values.config.backendAgent | quote }}
            - name: OPENAB_AGENT_NAME
              value: {{ .Values.config.agentName | quote }}
            - name: GHPOOL_URL
              value: {{ .Values.config.ghpoolUrl | quote }}
            - name: GHPOOL_API_URL
              value: {{ .Values.config.ghpoolUrl | quote }}
            - name: STATE_BUCKET
              value: {{ .Values.config.stateBucket | quote }}
          envFrom:
            - secretRef:
                name: {{ include "openab-bot.fullname" . }}-secrets
          resources:
            requests:
              cpu: {{ .Values.resources.cpu }}
              memory: {{ .Values.resources.memory }}
            limits:
              cpu: {{ .Values.resources.cpu }}
              memory: {{ .Values.resources.memory }}
```

That's the whole chart — 6 files, ~80 lines of YAML. Roughly the same size as
the equivalent `service.yaml` Pahud uses with ecsctl.

## Spawning N clones

The exact ergonomics Pahud's screenshot shows, but on GKE:

```bash
# Bot 1
helm install openab-chaodu ./openab-bot \
  --set name=chaodu \
  --set secrets.discordTokenKey=CHAODU_BOT_TOKEN

# Bot 2
helm install openab-chaodu2 ./openab-bot \
  --set name=chaodu2 \
  --set secrets.discordTokenKey=CHAODU2_BOT_TOKEN

# Bot 3 — different model, different state bucket
helm install openab-chaodu3 ./openab-bot \
  --set name=chaodu3 \
  --set secrets.discordTokenKey=CHAODU3_BOT_TOKEN \
  --set config.stateBucket=openab-state-experiment \
  --set image.tag=experimental

# See them all
helm list

# Tail one
kubectl logs -f -l app=openab-chaodu2

# Get into one
kubectl exec -it deployment/openab-chaodu2 -- bash

# Roll out a config change to one bot
helm upgrade openab-chaodu2 ./openab-bot \
  --set config.ghpoolUrl=http://ghpool-staging.openab.local:8080 \
  --reuse-values

# Delete one
helm uninstall openab-chaodu2
```

`helm list` shows all releases, `helm history` shows revisions per release, and
`helm rollback openab-chaodu2 3` reverts to revision 3 if a change breaks the
bot. These three capabilities are *not* present in ecsctl as of v0.3.0.

## Cost comparison: 10 always-on Discord bot clones

Reasonable assumptions: each clone needs 0.25 vCPU and 0.5 GB RAM continuously
(idle WebSocket connection + occasional STT inference).

| Platform | Configuration | Monthly cost (USD) | Notes |
|---|---|---|---|
| AWS Fargate Spot | `desiredCount: 1`, Fargate Spot | **~$25-30** | Pahud's choice; 70% discount on capacity |
| AWS Fargate on-demand | same, no Spot | ~$80 | No Spot pricing |
| **GCP GKE Autopilot** | 10 pods, 0.25 vCPU + 0.5 GB each | **~$30** | Cluster fee ~$73/mo offset by $74/mo free credit → effectively $0 |
| GCP Cloud Run | `--min-instances=1 --no-cpu-throttling` | ~$40-50 | No Spot equivalent; "CPU always allocated" billed continuously |
| GCP Compute Engine MIG (Spot) | container-vm template | ~$15 | Cheapest but the most operational overhead |

The headline: **AWS Fargate Spot and GKE Autopilot are within ~$5 of each other
for this workload**. The AWS Spot discount is meaningful, but the GKE Autopilot
free-tier offsets cluster cost. Cloud Run is the most ergonomic option on GCP
but the most expensive of the serverless tier because it lacks a Spot equivalent.

## When ecsctl beats Helm

There are three situations where ecsctl is the better choice even on AWS:

1. **You're committed to ECS for governance reasons.** Some enterprises have
   approved-runtime lists that include ECS but not EKS. ecsctl gives you a
   familiar declarative UX without introducing a new control plane.
2. **You need Fargate Spot.** GKE Autopilot has no Spot equivalent. If the
   workload runs continuously and is interruption-tolerant, Spot is real money.
3. **Your team already runs ECS in production and adding k8s is overhead.** Even
   though Helm is more mature, adopting it means adopting Kubernetes — control
   plane, CNI, RBAC, the whole package. ecsctl avoids that.

## When Helm beats ecsctl

Almost everywhere else:

1. **You're on GCP, Azure, or any multi-cloud k8s** — ecsctl only targets ECS.
2. **You need `helm rollback`, `helm history`, or `helm diff`** — features
   ecsctl doesn't have yet (v0.3.0).
3. **Your team already knows k8s** — Helm is built-in muscle memory.
4. **You want template loops, conditionals, or includes** — Go templates handle
   this; ecsctl's `--set` is flat.
5. **You want dependency management** — `Chart.yaml` `dependencies:` block lets
   you compose charts. ecsctl has no equivalent.

## A note on protocol convergence

The deeper trend here is that **the "clone-via-template" abstraction has
finally become cloud-runtime-portable**:

- Helm formalized it for k8s in 2016.
- Pulumi and Crossplane extended it to multi-cloud control planes.
- ecsctl, in 2026, brought it to ECS — the largest holdout in the major cloud
  serverless-container market without this pattern.

If you're choosing where to invest skill development, **Helm is the more
durable bet**. It's the abstraction every other tool is converging toward.

## See also

- [oablab/ecsctl](https://github.com/oablab/ecsctl) — the project this document is responding to
- [comparison-3-projects.md](./comparison-3-projects.md) — broader analysis of openab and its ECS Control Plane bet
- [providers.md](./providers.md) — how ggcoder uses Anthropic's `@anthropic-ai/sdk` to spoof Claude Code identity for subscription auth
- [Helm Charts Tutorial](https://helm.sh/docs/chart_template_guide/) — official Helm template guide
- [External Secrets Operator](https://external-secrets.io/) — for the GCP Secret Manager / AWS Secrets Manager bridge with `property:` JSON-key support
