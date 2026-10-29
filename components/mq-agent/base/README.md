## MQ Agent

Deploys the [IBM MQ Agent](https://www.ibm.com/docs/en/ibm-mq/9.4.x?topic=mq-agent) into the `cp4i` namespace via the `ibm-mq-agent` Helm chart. The MQ Agent provides an AI-powered chat assistant that connects to your queue manager (`NATIVEHAQM`) to answer diagnostic and operational questions.

### Files

| File | Purpose |
|---|---|
| `ccdt-configmap.yaml` | CCDT ConfigMap — tells the MQ Agent MCP server how to connect to `NATIVEHAQM` over channel `HAQMCHL` |
| `overrides.yaml` | Helm chart values (non-sensitive) — committed to git |
| `overrides-secrets.yaml.template` | Template for the local secrets override file |
| `kustomization.yaml` | Kustomize manifest for the CCDT ConfigMap (applied by ArgoCD) |

### Secrets you must create manually before deploying

The following Secrets must be created in the `cp4i` namespace. They contain sensitive data and are **never stored in git**.

#### 1. Image pull secret (already exists if MQ is installed)

```bash
oc create secret docker-registry ibm-entitlement-key \
  --docker-server=cp.icr.io \
  --docker-username=cp \
  --docker-password=<your-ibm-entitlement-key> \
  --namespace=cp4i
```

#### 2. TLS secret for MQ Console ↔ Agent communication

```bash
oc create secret tls mq-agent-tls \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=cp4i
```

### Deploying the Helm chart with secrets

Because the IBM MQ Agent Helm chart takes the watsonx `projectID` and `key` as plain values (and creates its own internal Secret from them), they cannot be supplied as a pre-existing cluster Secret. Instead:

1. Copy the template:

   ```bash
   cp components/mq-agent/base/overrides-secrets.yaml.template \
      components/mq-agent/base/overrides-secrets.yaml
   ```

2. Edit `overrides-secrets.yaml` and fill in your values:

   ```yaml
   agent:
     model:
       providers:
         myWatsonxRuntime:
           watsonxService:
             projectID: <your-watsonx-project-id>
             key: <your-ibm-cloud-api-key>
   ```

3. Add the IBM Helm repo (once):

   ```bash
   helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm
   helm repo update ibm-helm
   ```

4. Deploy:

   ```bash
   helm install ibm-mq-agent ibm-helm/ibm-mq-agent \
     --namespace cp4i \
     --values components/mq-agent/base/overrides.yaml \
     --values components/mq-agent/base/overrides-secrets.yaml
   ```

   `overrides-secrets.yaml` is git-ignored and never committed.

### ArgoCD

The CCDT ConfigMap (`mq-agent-config`, sync-wave 350) is managed by ArgoCD from this directory.

The Helm chart (`mq-agent`, sync-wave 400) is also declared as an ArgoCD Application in [`argocd/operands/mq-agent.yaml`](../../../argocd/operands/mq-agent.yaml) using ArgoCD's multi-source support. ArgoCD handles the non-sensitive `overrides.yaml`; the secrets values must be injected at initial `helm install` time as described above. On subsequent upgrades use `helm upgrade` with both values files.

### Queue manager access

Your queue manager (`NATIVEHAQM`) has `MQSNOAUT=yes` set, so no `setmqaut` grants are required. If you remove that setting in future, run:

```bash
setmqaut -m NATIVEHAQM -t qmgr                               -g mqagent +connect +ctrl
setmqaut -m NATIVEHAQM -n SYSTEM.ADMIN.COMMAND.QUEUE  -t queue -g mqagent +dsp +inq +put
setmqaut -m NATIVEHAQM -n SYSTEM.DEFAULT.MODEL.QUEUE  -t queue -g mqagent +dsp +get
setmqaut -m NATIVEHAQM -n SYSTEM.DEAD.LETTER.QUEUE    -t queue -g mqagent +dsp +browse
```
