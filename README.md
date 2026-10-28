# adp-backstage-scaffold

Backstage scaffolder templates for the ITGix Application Development Platform (ADP). A template renders a skeleton of Helm charts and ArgoCD Application manifests and opens a pull request (GitHub) or merge request (GitLab) against the environment's applications GitOps repository. ArgoCD applies the change after the merge.

## Templates

All templates live in `argo-helm-templates/`.

- `bckstg-scaf-templ-argocd-app-github.yaml` - new application (Helm chart plus ArgoCD Application), pull request on GitHub.
- `bckstg-scaf-templ-argocd-app-gitlab.yaml` - the same, merge request on GitLab.
- `bckstg-scaf-templ-devlake.yaml` - Apache DevLake as an optional platform tool, GitHub or GitLab chosen by a parameter.
- `bckstg-scaf-templ-pyroscope.yaml` - Grafana Pyroscope as an optional platform tool, with cloud-aware object storage (AWS S3 or GCP GCS).
- `bckstg-scaf-templ-alloy.yaml` - Grafana Alloy as an optional platform tool, installed with the upstream chart defaults.
- `bckstg-scaf-templ-nexus.yaml` - Sonatype Nexus Repository as an optional platform tool (artifact repository and proxy cache).

The first two are scaffolder forms, registered individually in Backstage; the rest are optional platform applications listed in `catalog-info.yaml`. Other files in the directory (the Phoenix variant, the dynamic environment variables template, the bulk version update and the DevOps agent request) are not registered in the catalog.

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
## Pyroscope

Template `pyroscope` ("Grafana Pyroscope") installs Grafana Pyroscope (continuous profiling) into the environment's applications repository. The chart is pinned to the upstream Pyroscope chart `2.3.1`.

The template is cloud-aware: the storage backend and the service-account identity are chosen from the environment's infra-facts at sync time, not in the form. The user supplies only a bucket name and a workload-identity name that work for either cloud.

Prerequisites in the target environment:

- An existing object storage bucket for profiling data (an S3 bucket on AWS, a GCS bucket on GCP).
- A pre-created cloud identity for the Pyroscope service account to access that bucket:
  - AWS: an IAM role with a trust policy allowing the `system:serviceaccount:<namespace>:pyroscope` subject through the cluster's OIDC provider (IRSA), and a bucket policy granting access.
  - GCP: a Google service account bound to the Kubernetes service account via Workload Identity, with access to the GCS bucket.
- Infra-facts values in the applications repository providing `cloud`, `region`, `applications_destination_repo`, and the account identifier used to build the role ARN (`aws_account_id` on AWS) or the GCP project (`project_id` / `gcp_project_id` on GCP). An optional `storage_class_name` fact overrides the per-cloud default PVC storage class (`gp3` on AWS, `standard-rwo` on GCP).

Inputs: `stage` and `region` (required), `namespace` (default `monitoring`), `bucket_name` and `workload_identity_name` (required), and `git_provider` (`github` or `gitlab`, default `github`). The workload-identity name is the IAM role name on AWS or the Google service account name/email on GCP.

The pull or merge request (branch `feat/pyroscope`) adds to the applications repository:

- `helm/pyroscope/` - wrapper chart around the upstream Pyroscope chart. The base `values.yaml` holds only cloud-neutral settings (agent and minio disabled, `persistence.enabled`, service account name).
- `helm/pyroscope/values/<stage>/<region>/values.yaml` - per-environment configuration (`replicaCount`, `persistence.size`, `resources`).
- `helm/applications/templates/application-pyroscope.yaml` - the ArgoCD Application (namespace from the `namespace` input). At sync time it injects, per cloud: the storage backend (`s3` with endpoint/region/SSE on AWS, `gcs` on GCP), the service-account annotation (`eks.amazonaws.com/role-arn` on AWS, `iam.gke.io/gcp-service-account` on GCP), and the PVC storage class.

The Application renders unless `app_pyroscope_enabled` is set to `false` in the environment's infra-facts values (set it in the installer config YAML); remove the file or set the switch to uninstall.

## Alloy

Template `alloy` ("Grafana Alloy") installs Grafana Alloy into the environment's applications repository using the upstream chart defaults. The chart is pinned to the upstream Alloy chart `1.13.0`.

By default the chart runs Alloy as a DaemonSet with an empty pipeline. It starts cleanly but does not collect or forward any telemetry until you provide an Alloy configuration (the chart's `alloy.configMap.content` value), most naturally in the per-environment values file.

Prerequisites in the target environment:

- Infra-facts values in the applications repository providing `applications_destination_repo`.

Inputs: `stage` and `region` (required), `namespace` (default `monitoring`), and `git_provider` (`github` or `gitlab`, default `github`).

The pull or merge request (branch `feat/alloy`) adds to the applications repository:

- `helm/alloy/` - wrapper chart around the upstream Alloy chart. The base `values.yaml` keeps the chart defaults (`alloy: {}`).
- `helm/alloy/values/<stage>/<region>/values.yaml` - per-environment overrides (empty stub by default).
- `helm/applications/templates/application-alloy.yaml` - the ArgoCD Application (namespace from the `namespace` input). No per-cloud values are injected; the chart defaults flow through unchanged.

The Application renders unless `app_alloy_enabled` is set to `false` in the environment's infra-facts values (set it in the installer config YAML); remove the file or set the switch to uninstall.

## Nexus Repository

Template `nexus-repository` ("Nexus Repository") installs Sonatype Nexus Repository (Community Edition) into the environment's applications repository. It is a self-hosted artifact repository with hosted, proxy and group repositories for Maven, npm, PyPI, Docker, Helm and raw files. The chart is pinned to the community chart `stevehipwell/nexus3` `5.27.0` (Nexus `3.97.0`).

Prerequisites in the target environment:

- The ingress controller (AWS Load Balancer Controller on AWS, GCE ingress on GCP) and a CSI driver with the `gp3` (AWS) or `standard-rwo` (GCP) storage class.
- Infra-facts values in the applications repository providing `cloud`, `project_name`, `region`, `environment`, `dns_main_domain` and `applications_destination_repo`.

Inputs: `stage` and `region` (required), `hostname` (optional; empty means `<project_name>-<region>-nexus-<environment>.<dns_main_domain>`), sizing (`storage_size` default `50Gi`, `cpu_request` `500m`, `memory_request` `2Gi`, `cpu_limit` `2`, `memory_limit` `4Gi`, `java_heap` `2048m`) and `git_provider` (`github` or `gitlab`, default `github`).

The pull or merge request (branch `feat/nexus`) adds to the applications repository:

- `helm/nexus/` - wrapper chart around the `nexus3` chart (persistence on, ingress on), and on GCP the `ManagedCertificate` and `FrontendConfig`.
- `helm/nexus/values/<stage>/<region>/values.yaml` - per-environment sizing from the form (PVC size, resources, JVM heap).
- `helm/applications/templates/application-nexus.yaml` - the ArgoCD Application (namespace `nexus`). It injects the ingress class, annotations (including the ALB health check path `/service/rest/v1/status` on AWS) and host, and the PVC storage class per cloud.

The Application renders unless `app_nexus_enabled` is set to `false` in the environment's infra-facts values (set it in the installer config YAML); remove the file or set the switch to uninstall.

First sign-in: user `admin`; Nexus generates the initial password on first start. Read it once with `kubectl -n nexus exec nexus-0 -- cat /nexus-data/admin.password` and change it in the setup wizard.

Notes:

- Community Edition uses the embedded database and has usage limits; check Sonatype's current terms before heavy use.
- Docker hosted/group repositories on dedicated connector ports are not exposed by this template.
- The PVC size can only be grown, and the StatefulSet volume claim is immutable, so a later size change needs a manual PVC resize.
