# adp-backstage-scaffold

Backstage scaffolder templates for the ITGix Application Development Platform (ADP). A template renders a skeleton of Helm charts and ArgoCD Application manifests and opens a pull request (GitHub) or merge request (GitLab) against the environment's applications GitOps repository. ArgoCD applies the change after the merge.

## Templates

All templates live in `argo-helm-templates/`.

- `bckstg-scaf-templ-argocd-app-github.yaml` - new application (Helm chart plus ArgoCD Application), pull request on GitHub.
- `bckstg-scaf-templ-argocd-app-gitlab.yaml` - the same, merge request on GitLab.
- `bckstg-scaf-templ-devlake.yaml` - Apache DevLake as an optional platform tool, GitHub or GitLab chosen by a parameter.
- `bckstg-scaf-templ-pyroscope.yaml` - Grafana Pyroscope as an optional platform tool, with cloud-aware object storage (AWS S3 or GCP GCS).
- `bckstg-scaf-templ-alloy.yaml` - Grafana Alloy as an optional platform tool, installed with the upstream chart defaults.
- `bckstg-scaf-templ-sonarqube.yaml` - SonarQube Server as an optional platform tool (code quality and security analysis), with a CloudNativePG or external PostgreSQL database.

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

## SonarQube

Template `sonarqube` ("SonarQube") installs SonarQube Server into the environment's applications repository. The chart is pinned to the official SonarSource chart `sonarqube/sonarqube` `2026.5.1004` (SonarQube Server 2026.5 LTA) from `https://SonarSource.github.io/helm-chart-sonarqube`. Editions and images:

- Community Build (default): `sonarqube:26.9.0.129388-community`.
- Developer Edition: `sonarqube:2026.5.4-developer`.
- Enterprise Edition: `sonarqube:2026.5.4-enterprise`.

Prerequisites in the target environment (AWS only):

- The AWS Load Balancer Controller and the `gp3` storage class.
- The External Secrets Operator with the `secretstore-aws` ClusterSecretStore (both database modes; the monitoring passcode is generated by an ESO `Password` generator).
- Database mode `cnpg`: the cloudnative-pg operator (`enable_cloudnative_pg: true` in the installer config).
- Database mode `external`: an existing PostgreSQL 14-18 database with UTF-8 encoding, owned by a dedicated SonarQube user, an AWS Secrets Manager secret holding that user's JSON `{"username": "...", "password": "..."}` with rotation off, and TLS on the server (the JDBC URL uses `sslmode=require`). Do not use the RDS-managed master secret (`rds!cluster-...`): it rotates every 7 days by default, and SonarQube reads the password only at pod start, so new DB connections fail after a rotation until the pod restarts. The master user would also give SonarQube superuser rights.
- Infra-facts values in the applications repository providing `project_name`, `region`, `environment`, `dns_main_domain` and `applications_destination_repo`.

Inputs:

- `stage` and `region` (required), `hostname` (optional; empty means `<project_name>-<region>-sonarqube-<environment>.<dns_main_domain>`), `ingress_enabled` (default `true`) and `edition` (`community`, `developer` or `enterprise`, default `community`).
- `db_mode`: `cnpg` (default) or `external`. For `cnpg`: `db_instances` (1-3, default 1) and `db_storage_size` (default `10Gi`). For `external`: `db_host`, `db_port` (default `5432`), `db_name` (default `sonarqube`) and `db_credentials_remote_key` (the full Secrets Manager secret name). The form only shows the fields of the selected mode.
- Sizing: `storage_size` default `20Gi`, `cpu_request` `500m`, `cpu_limit` `2`, `memory_request` `6Gi`, `memory_limit` `6Gi` (keep request = limit).
- `git_provider` (`github` or `gitlab`, default `github`).

The pull or merge request (branch `feat/sonarqube`) adds to the applications repository:

- `helm/sonarqube/` - wrapper chart around the `sonarqube` chart: persistence on `gp3`, ALB ingress with the health check path `/api/system/status`, the passcode and admin password `Password` generators and `ExternalSecret`s (`sonarqube-passcode`, `sonarqube-admin`), and depending on `database.mode` either the CNPG `Cluster` `sonarqube-db` or the `ExternalSecret` `sonarqube-db-credentials`. They use sync-wave `-1` (the generators `-2`), so they exist before SonarQube.
- `helm/sonarqube/values/<stage>/<region>/values.yaml` - per-environment edition, database mode and JDBC settings, PVC size and resources.
- `helm/applications/templates/application-sonarqube.yaml` - the ArgoCD Application (namespace `sonarqube`). It only sets the ingress: class `alb` and the ALB annotations (internet-facing, target-type ip, HTTP/HTTPS with redirect, health check `/api/system/status`, stickiness), merged like DevLake with the infra-facts keys `ingress_class_name`/`aws_ingress_class_name` and `ingress_annotations`/`aws_ingress_annotations` (the infra-facts values win). Env-wide settings such as a certificate ARN, WAF ACL or `group.name` apply to SonarQube too, and `alb.ingress.kubernetes.io/scheme: internal` there makes SonarQube internal. It does not set `Replace=true` (unlike the other Applications), because `kubectl replace` would drop fields that controllers set on the Ingress, CNPG and ESO objects. It uses `SkipDryRunOnMissingResource=true` and a sync retry, because the CNPG and ESO CRDs come from other Applications.

The Application renders unless `app_sonarqube_enabled` is set to `false` in the environment's infra-facts values (set it in the installer config YAML); remove the file or set the switch to uninstall.

Database modes:

- `cnpg`: CNPG creates Service `sonarqube-db-rw` and Secret `sonarqube-db-app`; SonarQube connects to `jdbc:postgresql://sonarqube-db-rw:5432/sonarqube` with the username and password from that Secret. No CNPG backups are configured. The names follow `database.cnpg.clusterName` (`sonarqube-db`); if you change it, change `jdbcOverwrite.jdbcUrl`, `jdbcOverwrite.jdbcSecretName` and the `SONAR_JDBC_USERNAME` `secretKeyRef` in the per-environment values with it.
- `external`: the `ExternalSecret` copies `username` and `password` from the Secrets Manager secret into Secret `sonarqube-db-credentials` (refreshed hourly). SonarQube connects to `jdbc:postgresql://<db_host>:<db_port>/<db_name>?sslmode=require` (TLS mandatory, no plaintext fallback; the server certificate is not verified).

In both modes the JDBC username is read from the Secret through the `SONAR_JDBC_USERNAME` env var, which overrides the placeholder `jdbcOverwrite.jdbcUsername`. No password is stored in git.

Admin login: user `admin` with a password generated in-cluster by an ESO `Password` generator into Secret `sonarqube-admin` (keys `password` and `currentPassword: admin`), never stored in git. The chart's change-admin-password hook Job (`setAdminPassword.passwordSecretName`, an Argo CD PostSync hook) replaces the default `admin`/`admin` as soon as SonarQube reports UP. Read the password with:
`kubectl -n sonarqube get secret sonarqube-admin -o jsonpath='{.data.password}' | base64 -d`
The Secret is generated once (`refreshInterval: "0"`). If you change the admin password in the UI, the Secret is no longer current; the hook Job on later syncs then fails the change quietly and does nothing. Deleting the `ExternalSecret` generates a new value that will not match the live password.

Developer and Enterprise Edition need a license key. Enter it in the SonarQube UI after install (Administration > Configuration > License Manager). Pick the edition at install time: Community Build is numbered ahead of SonarQube Server, so switching from Community to Developer/Enterprise later is a database downgrade that SonarQube rejects.

Notes:

- Elasticsearch needs `vm.max_map_count` >= 524288. The chart's `init-sysctl` init container (privileged, root) raises it, `fs.file-max` and the ulimits on the node, and only if they are lower. The platform's Kyverno policies (Audit mode, best practices only) do not block it. To check a node: `kubectl debug node/<node> -it --image=busybox -- sysctl vm.max_map_count`. If the nodes are tuned and a pod-security-baseline policy forbids privileged containers, set `sonarqube.initSysctl.enabled: false` in the per-environment values.
- The `init-fs` init container (root) chowns the data directories to SonarQube's uid 1000 / gid 0, because a new EBS volume is root-owned.
- The PVC is a plain `PersistentVolumeClaim` (`sonarqube`), so `persistence.size` can be grown later in the per-environment values.
- The ALB is internet-facing unless infra-facts set another scheme (see above).
