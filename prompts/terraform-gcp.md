# Stack: Terraform on Google Cloud (google and google-beta providers)

A general review prompt for Terraform that manages Google Cloud: root modules per environment, shared child modules, and remote state in a Cloud Storage bucket. Reviewers should apply these conventions in addition to the base review areas. Project-specific rules (the state bucket, naming scheme, allowed regions, which roles a team may grant) should be supplied via the consumer's `extra_prompt` or `extra_prompt_path` input and will appear under the "Repo-specific notes" section. When such a note exists, prefer it over the guidance below.

Several base-prompt bullets are application-shaped and do not apply here: `any` / `unknown`, floating promises, N+1 queries and request validation. Read the diff as a change to live infrastructure. A line that looks small can delete a database or open a service to the internet, so judge each change by what `terraform apply` would do to resources that already exist.

## IAM

- **Primitive roles.** `roles/owner`, `roles/editor` and `roles/viewer` grant far more than any workload needs. Flag them on any member that is not a break-glass human group, and name the predefined role that covers the actual use.
- **Grant at the narrowest resource.** A project-level `google_project_iam_member` for a service account that reads one bucket or one secret should be `google_storage_bucket_iam_member` or `google_secret_manager_secret_iam_member` on that resource. Flag project-wide grants where a resource grant would do.
- **`_policy` and `_binding` are authoritative.** `google_project_iam_policy` replaces the whole project policy, including grants made by Google for service agents; a mistake here can lock everyone out. `google_*_iam_binding` owns every member of its role, so any member added outside this config, or in another module, is removed on the next apply. Prefer `_member`. When `_binding` is intended, the members list must be complete, and only one place in the codebase may manage that role on that resource.
- **Do not mix the three kinds on one role and resource.** `_policy` with `_binding` or `_member`, or `_binding` with `_member` for the same role, fight each other and the plan never settles.
- **Service account keys.** `google_service_account_key` puts a long-lived private key in state. Flag it unless the comment says why workload identity federation or attached service accounts cannot be used.
- **Impersonation and actAs.** `roles/iam.serviceAccountUser` and `roles/iam.serviceAccountTokenCreator` on a service account with broad roles let the grantee act as it. Treat these grants with the same care as the roles that account holds.
- **Workload identity federation conditions.** A provider whose `attribute_condition` is missing or only checks the owner admits every repository or workload in that owner. Check that the condition names the exact repository, and the ref or workflow where the design calls for it.
- **Basic and legacy grants on buckets.** `roles/storage.legacyBucketOwner` and ACL resources (`google_storage_bucket_acl`) bypass uniform bucket-level access. Flag them on new buckets.

## Secrets in state and plan output

- **State holds every attribute in plain text,** including those marked `sensitive`. A secret value written in Terraform (`google_secret_manager_secret_version.secret_data`, a `random_password`, a database user password, a service account key) is readable by anyone who can read the state bucket. Flag new secret values created in Terraform and ask whether the secret can be created outside it, with Terraform managing only the secret container and its IAM.
- **Variables and outputs carrying secrets must be `sensitive = true`,** or the value prints in plan output and CI logs. An output of a secret should not exist unless a caller needs it.
- **No secret defaults.** A `default` on a password or token variable commits the value to the repository.
- **`.tfvars` files with real values** must not be committed. Flag a new `*.tfvars` holding anything that looks like a key, token or password.
- **The state bucket itself** needs versioning, uniform bucket-level access, public access prevention and a narrow IAM list. Flag changes that weaken any of them.

## Public exposure

- **Allow rules open to `0.0.0.0/0`** (or `::/0`). Backends behind a proxy-based Application Load Balancer receive traffic only from Google's front ends and health checks (`130.211.0.0/22`, `35.191.0.0/16`), so their allow rule names those ranges; an open rule on 80 or 443 exposes any backend with an external address directly. Only a passthrough Network Load Balancer, which keeps client addresses, needs an open source range, and the rule should say so. SSH and RDP from the internet should go through IAP (`35.235.240.0/20`) instead. A `deny` rule from `0.0.0.0/0` at low priority is a safe floor, not a finding. Check `direction`, `source_ranges` and `target_tags`; a rule with no target applies to every instance in the network.
- **`allUsers` and `allAuthenticatedUsers`** in any IAM member. `allAuthenticatedUsers` means any Google account, not users of this organisation. On a Cloud Run service `allUsers` with `roles/run.invoker` is a deliberate public endpoint; ask for a comment saying so.
- **Public buckets.** A bucket without `public_access_prevention = "enforced"`, or with `uniform_bucket_level_access = false`, can be made public by an ACL. Flag both on new buckets unless the bucket serves public content and says so.
- **Cloud SQL public IP.** `ipv4_enabled = true` with `authorized_networks` containing `0.0.0.0/0` exposes the database to the internet. Prefer private IP; if public IP stays, require SSL (`ssl_mode`) and narrow networks.
- **Ingress settings.** Cloud Run `ingress = "INGRESS_TRAFFIC_ALL"` and Cloud Functions with `ALLOW_ALL` are reachable from anywhere; check that matches the intent when a load balancer or IAP is meant to sit in front.
- **External IPs on instances** (`access_config` blocks) that do not need one. Outbound access belongs to Cloud NAT.

## Destroy and recreate on stateful resources

- **Read every `ForceNew` change.** Renaming a Cloud SQL instance, a bucket, a Filestore share, a Memorystore instance, a disk or a KMS key ring, or changing an immutable field such as `region` or a disk `type`, replaces the resource and loses its data. Ask what the plan shows: `must be replaced` on a stateful resource is a finding unless the description says the data is disposable.
- **Protection flags.** Stateful resources should carry `deletion_protection = true` (Cloud SQL, GKE, Spanner, Bigtable, BigQuery tables) or `lifecycle { prevent_destroy = true }`. Flag removal of either in the same change that also touches the resource.
- **`force_destroy = true` on a bucket** deletes every object on destroy. Flag it on any bucket holding data that is not reproducible.
- **KMS keys cannot be deleted, only scheduled for destruction,** and data encrypted with a destroyed key version is lost. Flag `prevent_destroy` missing on `google_kms_crypto_key`.
- **`create_before_destroy`** is needed where a replacement would otherwise cause downtime (instance templates, certificates); check that names are unique enough for both to exist at once.

## Versions and modules

- **Pin Terraform and providers.** `required_version` and `required_providers` with a version constraint in every root module, and the `.terraform.lock.hcl` file committed. A root module without a lock file gets whatever provider version is newest on the day of the apply.
- **Constraints should be bounded.** `~> 6.0` or an exact version, not `>= 5.0`, which admits a future major version with breaking changes.
- **`google` and `google-beta` at the same version.** Mismatched versions of the two providers in one root module produce confusing diffs.
- **Module sources pinned.** A Git module source needs `?ref=` with a tag or commit; a registry module needs `version`. An unpinned module changes under the caller without a diff in this repository.
- **A provider upgrade in the same change as resource edits** makes the plan hard to read. Ask for the upgrade on its own when the diff is large.

## Resource addressing: count, for_each, moved and import

- **`for_each` over `count` for anything that is not identical copies.** With `count`, removing an item from the middle of a list shifts every later index and Terraform destroys and recreates those resources.
- **`for_each` keys must be stable and known at plan time.** Keys built from resource attributes that are computed on apply fail the plan. Keys built from list positions, display names that may change, or values from `timestamp()` or `uuid()` cause replacement when they change. Use a static identifier from the input map.
- **Changing a resource address destroys and recreates it** unless a `moved` block says where it went. Flag a renamed resource, a resource moved into a module, or a switch from `count` to `for_each` that has no `moved` block.
- **`import` blocks** for resources created outside Terraform, rather than a manual `terraform import` that leaves no record. After the import has been applied everywhere, the block can be removed in a later change.
- **`removed` blocks** (or `terraform state rm`) when a resource should leave state without being destroyed. Deleting the resource block alone destroys it.

## Organisation policy, APIs and ordering

- **APIs must be enabled before resources that use them.** A resource created in the same apply as its `google_project_service` needs `depends_on` on that service, or the first apply fails with "API has not been used in project". Check that `disable_on_destroy = false` is set where other configs rely on the API.
- **Organisation policy constraints apply before the resource.** A change that relies on an exception (an external IP, a service account key, a public bucket, a resource outside the allowed locations) needs the `google_org_policy_policy` override to exist first, and the override should be scoped to the one project or folder that needs it.
- **Service agents appear after API enablement.** Grants to a Google-managed service account (`service-<number>@gcp-sa-*.iam.gserviceaccount.com`) fail until the API is on; use `google_project_service_identity` (google-beta) or `depends_on` rather than retrying.
- **IAM is eventually consistent.** A resource that uses a grant made in the same apply may fail the first time. A `depends_on` on the IAM resource narrows this; a `time_sleep` should carry a comment saying why.
- **Provider-level `project` and `region`.** Resources that set `project` explicitly should agree with the provider block, and a module should take the project as an input rather than reading it from a data source of the caller's provider.

## Code quality

- **Variables carry `type` and `description`,** and `validation` blocks where a wrong value is easy to pass (regions, CIDR ranges, environment names).
- **No hard-coded project ids, project numbers, bucket names or emails** inside child modules. They are inputs.
- **Labels on every resource that supports them,** at least environment and owning team, so cost and ownership can be read without opening the code.
- **`terraform fmt` and `terraform validate` clean.** If CI runs `tflint` or a policy tool (Checkov, tfsec, Trivy), do not repeat what it already reports; flag a new suppression comment that has no reason.
- **Comments explain why.** A comment on a `prevent_destroy`, a firewall exception or a broad grant saying what depends on it is worth more than any other line in the file.
