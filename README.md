# adp-backstage-scaffold

Backstage scaffolder templates for the ITGix Application Development Platform (ADP). A template renders a skeleton of Helm charts and ArgoCD Application manifests and opens a pull request (GitHub) or merge request (GitLab) against the environment's applications GitOps repository. ArgoCD applies the change after the merge.

## Templates

All templates live in `argo-helm-templates/`.

- `bckstg-scaf-templ-argocd-app-github.yaml` - new application (Helm chart plus ArgoCD Application), pull request on GitHub.
- `bckstg-scaf-templ-argocd-app-gitlab.yaml` - the same, merge request on GitLab.
- `bckstg-scaf-templ-devlake.yaml` - Apache DevLake as an optional platform tool, GitHub or GitLab chosen by a parameter.

The first two are scaffolder forms, registered individually in Backstage; the third is an optional platform application listed in `catalog-info.yaml`. Other files in the directory (the Phoenix variant, the dynamic environment variables template, the bulk version update and the DevOps agent request) are not registered in the catalog.

## Registering the templates in Backstage

Two kinds of catalog entries are registered, in `catalog.locations` of the Backstage `app-config.yaml`:

- The scaffolder forms (`bckstg-scaf-templ-argocd-app-github.yaml`, `bckstg-scaf-templ-argocd-app-gitlab.yaml`) are registered one by one, each with `rules: [{allow: [Template]}]`.
- The optional platform applications are registered through one `Location`, `catalog-info.yaml` at the repository root, with `rules: [{allow: [Location, Template]}]`:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/itgix/adp-backstage-scaffold/blob/main/argo-helm-templates/bckstg-scaf-templ-argocd-app-github.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/itgix/adp-backstage-scaffold/blob/main/argo-helm-templates/bckstg-scaf-templ-argocd-app-gitlab.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/itgix/adp-backstage-scaffold/blob/main/catalog-info.yaml
      rules:
        - allow: [Location, Template]
```

## Adding a new catalog item

- A new optional platform application: add the template file and its skeleton directory under `argo-helm-templates/`, then add one line to `spec.targets` in `catalog-info.yaml`, for example `- ./argo-helm-templates/bckstg-scaf-templ-<name>.yaml`. No change in the Backstage configuration is needed.
- A new scaffolder form: add the template file and register it as its own `catalog.locations` entry in the Backstage configuration.

Backstage picks up the change on its next catalog refresh.

## DevLake

Template `devlake` ("Apache DevLake") installs DevLake into the environment's applications repository.

Prerequisites in the target environment:

- A cloud secret named `<region_short>-<environment>-<project_name>-devlake-encryption-secret`, created through the installer's `custom_secrets` setting.
- External Secrets Operator with the `secretstore-aws` (AWS) or `secretstore-gcp` (GCP) ClusterSecretStore.
- Grafana in the `monitoring` namespace (DevLake uses `http://grafana.monitoring.svc.cluster.local`).

Inputs: `stage` and `region` (required), `hostname` (optional; empty means `<project_name>-<region>-devlake-<environment>.<dns_main_domain>`) and `git_provider` (`github` or `gitlab`, default `github`).

The pull or merge request (branch `feat/devlake`) adds to the applications repository:

- `helm/devlake/` - wrapper chart around the upstream Apache DevLake chart, with the encryption-secret ExternalSecret, and on GCP the `ManagedCertificate` and `FrontendConfig`.
- `helm/devlake/values/<stage>/<region>/values.yaml` - per-environment overrides.
- `helm/applications/templates/application-devlake.yaml` - the ArgoCD Application (namespace `devlake`, ingress class and annotations computed for AWS or GCP from the shared infra-facts values).

The Application renders unless `app_devlake_enabled` is set to `false` in the environment's infra-facts values (set it in the installer config YAML); remove the file or set the switch to uninstall.
