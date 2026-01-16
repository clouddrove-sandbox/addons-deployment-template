Here’s a **ready-to-use `README.md`** you can place next to your shared workflow.
It explains what the template does and how other workflows can call it.

---

# Shared AKS Addon Helm Deployment Template

This repository contains a **reusable GitHub Actions workflow** that standardizes how Helm charts (AKS addons) are deployed to an Azure Kubernetes Service (AKS) cluster.

The workflow is designed to be called from other workflows using `workflow_call`, making it easy to deploy multiple addons (Grafana, Prometheus, etc.) in a consistent, secure way.

---

## ✨ What this template does

When invoked, the workflow will:

1. Authenticate to Azure using a service principal.
2. Set the Kubernetes context for the target AKS cluster.
3. Install required tools (`kubectl`, `helm`, `kubelogin`).
4. Render a Helm values file and replace secret tokens.
5. Optionally add a Helm repository.
6. Deploy or upgrade a Helm release using:

   ```bash
   helm upgrade --install
   ```
7. Clean up temporary files and secrets.

---

## 📁 Files involved

* **`.github/workflows/addons-deployment.yaml`**
  The shared workflow template.

* **Calling workflow**
  Any workflow that uses:

  ```yaml
  uses: ./.github/workflows/addons-deployment.yaml
  ```

---

## 🔧 Inputs

| Input               | Required | Description                                            |
| ------------------- | -------- | ------------------------------------------------------ |
| `environment_name`  | ✅        | GitHub Environment name (e.g., `dev`, `prod`).         |
| `aks_rg`            | ✅        | Resource group of the AKS cluster.                     |
| `aks_cluster_name`  | ✅        | AKS cluster name.                                      |
| `addon_name`        | ✅        | Logical name of the addon (grafana, prometheus, etc.). |
| `helm_release_name` | ✅        | Helm release name.                                     |
| `helm_namespace`    | ✅        | Kubernetes namespace for deployment.                   |
| `helm_chart`        | ✅        | Chart path or chart reference.                         |
| `helm_repo_name`    | ❌        | Helm repo name (if using remote charts).               |
| `helm_repo_url`     | ❌        | Helm repo URL.                                         |
| `helm_values_file`  | ✅        | Path to Helm values file.                              |
| `token_prefix`      | ❌        | Token prefix for value replacement (default `#{`).     |
| `token_suffix`      | ❌        | Token suffix for value replacement (default `}#`).     |

---

## 🔐 Secrets

| Secret              | Required | Description                                                      |
| ------------------- | -------- | ---------------------------------------------------------------- |
| `AZURE_CREDENTIALS` | ✅        | Azure service principal JSON.                                    |
| `token_env`         | ✅        | Environment variables containing secrets for token replacement.  |
| `helm_set_args`     | ❌        | Extra `--set` arguments for Helm (e.g. `--set image.tag=1.2.3`). |

Example `token_env` secret:

```bash
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=supersecret
```

These variables are injected at runtime and used to replace tokens in the Helm values file.

---

## 🔄 How token replacement works

In your Helm values file, define placeholders:

```yaml
adminUser: "#{GRAFANA_ADMIN_USER}#"
adminPassword: "#{GRAFANA_ADMIN_PASSWORD}#"
```

At deploy time:

* Tokens are replaced using values from `token_env`.
* The rendered file is stored temporarily and deleted after deployment.

---

## ▶️ How to use this template from another workflow

### 1. Create a calling workflow

Example: `.github/workflows/deploy-addons.yaml`

```yaml
name: Deploy AKS Addon (Manual)

on:
  workflow_dispatch:
    inputs:
      deploy_grafana:
        description: "Deploy / upgrade Grafana?"
        required: true
        type: boolean
        default: false
      deploy_prometheus:
        description: "Deploy / upgrade Prometheus?"
        required: true
        type: boolean
        default: false
```

---

### 2. Call the shared workflow

#### Example: Deploy Grafana

```yaml
jobs:
  deploy_grafana:
    name: Deploy Grafana
    if: ${{ inputs.deploy_grafana == true }}
    uses: ./.github/workflows/addons-deployment.yaml

    with:
      environment_name: prod
      aks_rg: test-rg
      aks_cluster_name: test-aks

      addon_name: grafana
      helm_namespace: monitoring
      helm_release_name: grafana

      helm_chart: ./aks-addons/grafana
      helm_values_file: ./aks-addons/grafana/vars/override.yaml

    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      token_env: |
        GRAFANA_ADMIN_USER=${{ secrets.GRAFANA_ADMIN_USER }}
        GRAFANA_ADMIN_PASSWORD=${{ secrets.GRAFANA_ADMIN_PASSWORD }}
```

---

#### Example: Deploy Prometheus

```yaml
  deploy_prometheus:
    name: Deploy Prometheus
    if: ${{ inputs.deploy_prometheus == true }}
    uses: ./.github/workflows/addons-deployment.yaml

    with:
      environment_name: prod
      aks_rg: test-rg
      aks_cluster_name: test-aks

      addon_name: prometheus
      helm_namespace: monitoring
      helm_release_name: prometheus

      helm_chart: ./aks-addons/prometheus
      helm_values_file: ./aks-addons/prometheus/vars/override.yaml

    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      token_env: ${{ secrets.TOKEN_ENV }}
      helm_set_args: ${{ secrets.HELM_SET_ARGS }}
```

---

## 🌐 Using a remote Helm repo (optional)

If your chart is not local:

```yaml
with:
  helm_repo_name: prometheus-community
  helm_repo_url: https://prometheus-community.github.io/helm-charts
  helm_chart: prometheus-community/prometheus
```

The workflow will automatically:

```bash
helm repo add prometheus-community ...
helm repo update
```

---

## 🛡️ Environment protection

The workflow uses GitHub **Environments**:

```yaml
environment:
  name: ${{ inputs.environment_name }}
```

This allows you to:

* Enforce approvals for `prod`
* Scope secrets per environment
* Add deployment protection rules

---

## 🧪 Recommended repository structure

```text
.
├── .github/workflows/
│   ├── addons-deployment.yaml   # shared template
│   └── deploy-addons.yaml       # caller workflow
├── aks-addons/
│   ├── grafana/
│   │   ├── Chart.yaml
│   │   └── vars/override.yaml
│   └── prometheus/
│       ├── Chart.yaml
│       └── vars/override.yaml
```

---

## ✅ Best practices

* Keep **secrets only in GitHub Secrets**, never in values files.
* Use **token replacement** for sensitive fields.
* Use **GitHub Environments** for prod approvals.
* Keep addon-specific logic in the **calling workflow**, not in the shared template.
* Use `helm_set_args` only for small overrides; prefer values files for clarity.

---

## 📌 Summary

This shared workflow gives you:

* A **single, secure, repeatable** way to deploy Helm addons to AKS.
* Clear separation between:

  * **Platform logic** (template workflow)
  * **Addon configuration** (calling workflows)
* Easy extension for new addons by just adding another job that calls the template.

If you want, I can now:

* Turn this into a downloadable `README.md` file, or
* Add diagrams showing the workflow call flow.
