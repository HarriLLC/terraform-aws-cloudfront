# CLAUDE.md

## What This Module Provisions

- `aws_cloudfront_distribution.this` — the CDN distribution (created when `create_distribution = true`; uses `count`).
- `aws_cloudfront_origin_access_identity.this` — one per `origin_access_identities` map entry (only when `create_origin_access_identity = true`).
- `aws_cloudfront_origin_access_control.this` — one per `origin_access_control` map entry (only when `create_origin_access_control = true`).
- `aws_cloudfront_monitoring_subscription.this` — optional realtime CloudWatch metrics (when `create_monitoring_subscription = true`).
- Looks up existing cache / origin-request / response-headers policies by name via `data` sources, so behaviors can reference them without hardcoding IDs.

## Registry Source

```hcl
module "cdn" {
  source  = "app.terraform.io/harri/cloudfront/aws"
  version = "x.y.z"

  origin = {
    s3_one = {
      domain_name = "my-bucket.s3.amazonaws.com"
      s3_origin_config = { origin_access_identity = "s3_bucket_one" }
    }
  }

  default_cache_behavior = {
    target_origin_id       = "s3_one"
    viewer_protocol_policy = "redirect-to-https"
  }
}
```

> This is a vendored copy of the upstream `terraform-aws-modules/cloudfront/aws` module, republished to the Harri private registry. Pin `version` to an exact semver.

## Required Versions

| Component | Constraint |
|---|---|
| Terraform | `>= 0.13.1` |
| AWS provider | `>= 5.12.0` |

Source of truth: `versions.tf`. (Harri module standard is `terraform >= 1.9.0` / `aws >= 5.0` — see `.tf-init-notes.md`.)

## File Layout

```
terraform-aws-cloudfront/
├── variables.tf    # Input variables — type, description, defaults
├── main.tf         # locals + all resources + data sources (no resources.tf)
├── outputs.tf      # Values exposed to consumers
├── versions.tf     # required_version + required_providers
├── README.md       # Usage example, inputs/outputs reference
├── examples/       # Standalone example callers
│   └── complete/
└── wrappers/       # for_each wrapper around the root module
```

## Inputs

Selected inputs; full list in `variables.tf` / README.

| Name | Type | Default | Description |
|---|---|---|---|
| `create_distribution` | `bool` | `true` | Controls if the distribution is created. |
| `create_origin_access_identity` | `bool` | `false` | Create CloudFront OAIs from `origin_access_identities`. |
| `origin_access_identities` | `map(string)` | `{}` | Map of OAIs (value is the comment). |
| `create_origin_access_control` | `bool` | `false` | Create OACs from `origin_access_control`. |
| `origin_access_control` | `map(object)` | `{ s3 = {...} }` | Map of origin access controls. |
| `aliases` | `list(string)` | `null` | Alternate domain names (CNAMEs). |
| `comment` | `string` | `null` | Distribution comment. |
| `enabled` | `bool` | `true` | Whether the distribution accepts requests. |
| `http_version` | `string` | `"http2"` | Max HTTP version (`http1.1`/`http2`/`http2and3`/`http3`). |
| `is_ipv6_enabled` | `bool` | `null` | Enable IPv6. |
| `price_class` | `string` | `null` | `PriceClass_All`/`PriceClass_200`/`PriceClass_100`. |
| `retain_on_delete` | `bool` | `false` | Disable instead of delete on destroy. |
| `wait_for_deployment` | `bool` | `true` | Wait for `Deployed` status. |
| `web_acl_id` | `string` | `null` | WAF web ACL ID/ARN (Global region). |
| `staging` | `bool` | `false` | Whether this is a staging distribution. |
| `tags` | `map(string)` | `null` | Tags to assign. |
| `origin` | `any` | `null` | One or more origins. |
| `origin_group` | `any` | `{}` | One or more origin groups. |
| `viewer_certificate` | `any` | `{ cloudfront_default_certificate = true, ... }` | SSL configuration. |
| `geo_restriction` | `any` | `{}` | Geo restriction config. |
| `logging_config` | `any` | `{}` | Access-log config (max one). |
| `custom_error_response` | `any` | `{}` | Custom error responses. |
| `default_cache_behavior` | `any` | `null` | Default cache behavior. |
| `ordered_cache_behavior` | `any` | `[]` | Ordered cache behaviors (precedence top→bottom). |
| `create_monitoring_subscription` | `bool` | `false` | Create realtime monitoring subscription. |
| `realtime_metrics_subscription_status` | `string` | `"Enabled"` | `Enabled`/`Disabled` for realtime metrics. |

## Outputs

| Name | Description |
|---|---|
| `cloudfront_distribution_id` | Distribution identifier. |
| `cloudfront_distribution_arn` | Distribution ARN. |
| `cloudfront_distribution_domain_name` | Distribution domain name. |
| `cloudfront_distribution_status` | Current status (`Deployed` when propagated). |
| `cloudfront_distribution_hosted_zone_id` | Route 53 zone ID for alias records. |
| `cloudfront_distribution_etag` | Current version of the distribution info. |
| `cloudfront_distribution_caller_reference` | Internal caller reference. |
| `cloudfront_distribution_trusted_signers` | Active trusted signers (signed URLs). |
| `cloudfront_distribution_last_modified_time` | Last modified time. |
| `cloudfront_distribution_in_progress_validation_batches` | In-progress invalidation batches. |
| `cloudfront_distribution_tags` | All tags on the distribution. |
| `cloudfront_monitoring_subscription_id` | Monitoring subscription ID. |
| `cloudfront_origin_access_identities` | Map of created OAIs. |
| `cloudfront_origin_access_identity_ids` | IDs of created OAIs. |
| `cloudfront_origin_access_identity_iam_arns` | IAM ARNs of created OAIs. |
| `cloudfront_origin_access_controls` | Map of created OACs. |
| `cloudfront_origin_access_controls_ids` | IDs of created OACs. |

## Examples

- `examples/complete` — full distribution integrating S3 buckets, Lambda, CloudFront Functions, ACM cert, and Route53 records.

## Validate Changes Locally

- Run `terraform fmt` before opening a PR (always — CI may fail otherwise).
- From the module dir: `terraform init -upgrade=false`.
- Run `terraform validate`.
- **Do not run `terraform plan` or `terraform apply` locally** — modules don't hold state on their own; that happens in the consumer stack.

## Publishing a New Version

1. Make the changes; update `CHANGELOG.md` / `README.md` if behaviour changes.
2. Run `terraform fmt` and `terraform validate`.
3. Tag the commit: `git tag vX.Y.Z && git push --tags`.
4. HCP Registry auto-publishes from the tag; bump consumers' `version` pins.

## Conventions & Naming

### Project naming patterns

- All resources use the logical name `this`; multiplicity comes from `count` (distribution, monitoring subscription) or `for_each` (OAIs, OACs).
- OAIs and OACs are only created when both the `create_*` flag is true **and** the corresponding map is non-empty (see `locals` in `main.tf`).
- Cache behaviors resolve policies by name through `data.aws_cloudfront_cache_policy` / `origin_request_policy` / `response_headers_policy`; pass `*_policy_name` instead of `*_policy_id`.
- When a behavior uses a cache policy, set `use_forwarded_values = false` — otherwise CloudFront rejects `ForwardedValues`.

### Universal house style

- Variables: `type` + `description` required; snake_case; `nullable` declared explicitly. Use `optional(type, default)` for nested object fields.
- Module / component versions pinned to exact semver (e.g. `version = "4.1.1"`), not ranges.
- **Comments**: use `#` for both single-line and multi-line comments. Only comment to clarify non-obvious intent.
- **No hardcoded secrets**: never put credentials, tokens, or keys in Terraform files. Source them from env vars, TFC variable sets, or Vault (for modules, the consumer supplies them). Mark secret-holding variables with `sensitive = true`.
- **Indentation**: two spaces per nesting level (`terraform fmt` enforces — required before every PR).
- **Variable block field order**: `type`, `description`, `default`, `sensitive`, `validation`.
- **Output block field order**: `type`, `description`, `value`, `sensitive`.
- **Resource argument order**: `count`/`for_each` first, then a blank line, then non-block arguments, then block arguments, then `lifecycle`, then `depends_on`.
- **Blank lines within blocks**: separate logical groups of arguments with empty lines.
- **`count` vs `for_each`**: use `count` for nearly identical instances; use `for_each` when arguments differ per instance.
- **Tags — don't duplicate `default_tags`**: shared/common tags are applied once at the provider level via `default_tags { tags = local.common_tags }`, so they already land on every resource. **Never re-declare those same tags on individual resources or map entries** — only add `tags` to a resource for values that are genuinely resource-specific and not already in `common_tags`. Duplicating the common tags per-resource is redundant and drifts.
- **Data sources**: live in a separate `data.tf` file, logically positioned before the resources that reference them.
- **`.gitignore`**: exclude `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `*.tfplan`, and any `.tfvars` holding secrets. Keep `.terraform.lock.hcl` committed.

### Module-specific

- File set: `variables.tf`, `data.tf`, `locals.tf`, `resources.tf`, `outputs.tf`, `versions.tf`, `README.md`, `tests/`.
- Required versions standard: `terraform >= 1.9.0`, `aws >= 5.0`.
- Registry source convention: `app.terraform.io/harri/{name}/aws`.
- Publish: `git tag vX.Y.Z` → HCP Registry auto-publish.
- Tests: `terraform test` with `mock_provider "aws" {}`.
