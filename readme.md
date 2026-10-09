# Azure Infrastructure

Terraform for the Azure infrastructure behind [david-baker.co.uk](https://david-baker.co.uk), which comprises a personal website, Cricsheet API, and a Container Apps queue-scaling demo.

Container images are built and pushed by each application's own GitHub Actions workflow and pulled from **GitHub Container Registry**. This repository provisions infrastructure only; it does not build images.

---

## Layout

Two root modules, applied in order. `applications` reads `core`'s outputs via `terraform_remote_state`, so `core` must exist first.

```
Terraform/
├── core/            # long-lived, rarely changes
└── applications/    # the apps themselves, changes with releases
```

### `core`

| File | Resources |
|---|---|
| `resourcegroup.tf` | `rg-<product>` |
| `compute.tf` | App Service Plan `asp-<product>` (Linux, B1) |
| `networking.tf` | Virtual network `vnet-<product>` (subnets live in `applications`) |
| `documentdb.tf` | Cosmos DB account (free tier), document database and container |
| `servicebus.tf` | Service Bus namespace `sbns-<product>` (Basic, local auth disabled) |
| `containerappenvironment.tf` | Container App Environment `cae-<product>` |
| `dns.tf` | Public DNS zone for the apex domain |
| `oidc.tf` | User-assigned identity `github-oidc` + a federated credential per application repo |
| `outputs.tf` | Everything `applications` consumes |

### `applications`

| File | Resources |
|---|---|
| `main.tf` | `terraform_remote_state` data source pointing at `core` |
| `compute.tf` | Three Linux web apps (website, Cricsheet API, scaling API), their subnets, app settings and role assignments |
| `containerapps.tf` | Scaling worker container app, its user-assigned identity, the Service Bus queue and queue role assignments |
| `oidc.tf` | **Currently broken — see [Known issues](#known-issues)** |
| `outputs.tf` | Web app name |

---

## Prerequisites

- Terraform (the `required_version` constraint is `>= 0.12`, but the configuration uses `azurerm` 4.x features and should be run on a current 1.x release)
- Azure CLI, authenticated: `az login`
- A subscription with the target DNS zone delegated to Azure if you want working records
- Cosmos DB free tier is enabled on the account, so the subscription must not already have a free-tier account

State is **local** and `.tfstate` is gitignored. Both modules keep state in their own directory.

---

## Variables

Variables without defaults must be supplied. `*.tfvars` is gitignored, so create them locally.

### `Terraform/core/terraform.tfvars`

```hcl
product                       = "personal-website"
cric_api_cosmos_database_name = "cricsheet"
cric_api_cosmos_container_name = "matches"
personal_website_vnet_prefixes = ["10.0.0.0/16"]
```

Optional, with defaults worth knowing: `location` (`westeurope`), `app_service_plan_sku` (`B1`), `app_service_plan_kind` (`Linux`), `cric_api_partition_key` (`/id`), `dns_zone_name`, and the six `github_oidc_*` variables naming the organisation, repositories and branches that may request a token.

### `Terraform/applications/terraform.tfvars`

```hcl
product                                   = "personal-website"
web_app_repo_name                         = "personal_website"
cric_api_repo_name                        = "cricsheet_api"
cric_api_version                          = "v1"
scaling_name                              = "aca-scaling"
scaling_api_version                       = "v1"
personal_website_subnet_web_app_prefixes  = ["10.0.1.0/24"]
personal_website_subnet_cric_api_prefixes = ["10.0.2.0/24"]
```

Image tags all default to `latest`. The Cosmos role defaults to the built-in **Data Reader** (`00000000-0000-0000-0000-000000000001`); **Data Contributor** is `...0002`. Scaling behaviour is tunable via `scaling_worker_min_replicas` / `max_replicas` / `message_count` / `cooldown_period_in_seconds` / `interval_in_seconds`.

---

## Applying

```bash
cd Terraform/core
terraform init
terraform plan
terraform apply

cd ../applications
terraform init
terraform plan
terraform apply
```

Destroy in reverse: `applications` first, then `core`.

---

## How it fits together

**One B1 App Service Plan hosts all three web apps.** B1 is the floor for a custom domain with TLS, and it includes free App Service Managed Certificates.

**The two APIs are not publicly reachable.** Each sits in its own delegated subnet with VNet integration, and carries an `ip_restriction` pair: allow the website's subnet at priority 100, deny `0.0.0.0/0` at priority 200. The website is the only permitted caller. App Service Health Check still reaches them, because the platform probes internally.

**Images come from GHCR and Terraform does not own the tag.** Each web app and the container app declare an initial image, but

```hcl
lifecycle {
  ignore_changes = [site_config[0].application_stack[0].docker_image_name]
}
```

means a deployment pipeline can repoint the app at a new tag or digest without the next `terraform apply` reverting it.

**App settings are assembled from two locals per app** — a static base and a dynamic block that reads `core` outputs and sibling resources. The website's `MATCHES_API_BASE_URL` and `ACA_API_BASE_URL` are built from the APIs' `default_hostname`, so the wiring follows renames automatically.

**The scaling demo** is a Service Bus queue, a container app scaling 0→5 replicas on queue depth via a KEDA `azure-servicebus` rule, and an API that enqueues messages and reports replica counts.

---

## Access model

Everything authenticates with managed identity. Service Bus has `local_auth_enabled = false`, so SAS keys are not an option.

| Identity | Role | Scope |
|---|---|---|
| `github-oidc` (user-assigned) | Website Contributor | each of the three web apps |
| `github-oidc` | Container Apps Contributor | scaling container app |
| Cricsheet API (system-assigned) | Cosmos DB Data Reader | the Cosmos **container** |
| Scaling API (system-assigned) | Azure Service Bus Data Sender | the queue |
| Scaling API (system-assigned) | ContainerApp Reader | scaling container app |
| Scaling worker (system-assigned) | Azure Service Bus Data Receiver | the queue |
| `id-<scaling_name>-worker` (user-assigned) | Azure Service Bus Data Receiver | the queue |

The worker holds the receiver role twice by design: the **system-assigned** identity is used by application code to read messages, and a **user-assigned** identity is required by the scale rule, because a system-assigned principal does not exist at the point the rule is created.

GitHub's OIDC identity deliberately has no registry permission. Workflows authenticate to GHCR with the built-in `GITHUB_TOKEN`, so the Azure identity only needs enough rights to repoint and restart the apps.

---

## Known issues

**`Terraform/applications/oidc.tf` will fail.** It assigns `AcrPush` against `data.terraform_remote_state.core.outputs.container_registry.id`, but `core` has no container registry and no such output — it was removed when the images moved to GHCR. The file should be deleted.

**`Terraform/applications/acr-build-push.ps1` is a leftover** from the PowerShell era and no longer has a registry to push to.

**`.vscode/launch.json` configures a PowerShell debugger** and is no longer relevant.

**`scaling_api_image_tag` is declared but unused** — `compute.tf` uses `scaling_worker_image_tag` for the scaling API's image, so the worker and API tags cannot be set independently.

**The remote state path is `../Core/terraform.tfstate`** but the directory is `core`. This resolves on Windows and fails on a case-sensitive filesystem.

**`regex("https://(.*):d*", ...)`** is missing a backslash before `d`. It extracts the namespace host correctly by accident; `:\\d*` is what was intended.

**`providers.tf` in `applications` declares `azaapi`** (source `Azure/azapi`) and `random`, neither of which is used. The `azaapi` key is also a typo for `azapi`.

**`required_version = ">= 0.12"`** is loose enough to admit Terraform versions that cannot parse this configuration.

---

## Design notes

Observations worth keeping from building this out.

- **Cosmos DB role assignment scope needs the account name in front.** `dbs/<db>/colls/<coll>` is not enough; the scope must be `<account_id>/dbs/<db>/colls/<coll>`.
- **A private endpoint for Service Bus was considered and rejected** on cost for a hobby subscription. IP restrictions plus VNet integration cover the actual threat model.
- **DNS records are managed by hand.** Only the zone is in Terraform — records change rarely and a zone in state with manual records avoids churn.
- **`ip_restriction` is thinly documented.** The allow/deny priority ordering is the part to get right: a permissive rule at a lower priority number wins over a broad deny above it.
- **Microsoft's OIDC guidance says only "assign the appropriate role".** In practice it depends on what the workflow does — repointing and restarting a web app needs Website Contributor, a container app needs Container Apps Contributor, and image pull permissions belong to the app's own identity rather than the deployment identity.
- **ACR Tasks versus a GitHub runner:** building on the runner means the deployment identity never needs push rights at all, which is the least-privilege option and became moot once images moved to GHCR.
- **Naming follows** the [Cloud Adoption Framework resource abbreviations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations).

---

## Possible next steps

- Delete `oidc.tf`, `acr-build-push.ps1` and `.vscode/launch.json`
- Move state to an Azure Storage backend for locking, versioning and CI access
- Raise `required_version` to a realistic floor and drop the unused providers
- Pin `azurerm` more tightly than `~> 4.0` if plan stability matters
- Add `terraform fmt -check` and `terraform validate` as a GitHub Actions workflow
