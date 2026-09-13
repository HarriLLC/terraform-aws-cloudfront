# CLAUDE.md

`terraform-aws-cloudfront` — a reusable Harri Terraform **module**, published
to the HCP private registry and consumed by exact pinned version.

Harri's fork of `terraform-aws-modules/cloudfront`, republished to the HCP
private registry. Upstream structure and inputs are kept; Harri consumers drive
it from `TerraformCloudFront/common/distributions`.

## What This Module Provisions

- `aws_cloudfront_distribution` — one distribution with `default_cache_behavior`,
  `ordered_cache_behavior`, one or more `origin` / `origin_group` blocks,
  `viewer_certificate`, `geo_restriction`, `logging_config`, and
  `custom_error_response`.
- `aws_cloudfront_origin_access_control` per `origin_access_control` entry when
  `create_origin_access_control` (the modern S3-origin auth).
- `aws_cloudfront_origin_access_identity` per `origin_access_identities` entry
  when `create_origin_access_identity` (legacy S3-origin auth).
- `aws_cloudfront_monitoring_subscription` for additional real-time metrics
  when `create_monitoring_subscription`.

## Registry Source

```hcl
module "cloudfront" {
  source  = "app.terraform.io/harri/cloudfront/aws"
  version = "1.0.0"

  # no required inputs — set at least these in practice
  aliases                = ["assets.harriprep.com"]
  origin                 = { s3 = { domain_name = "..." } }
  default_cache_behavior = { target_origin_id = "s3", viewer_protocol_policy = "redirect-to-https" }
  viewer_certificate     = { acm_certificate_arn = "...", ssl_support_method = "sni-only" }
}
```

## Required Versions

| Component | Constraint |
|---|---|
| Terraform | `>= 0.13.1` |
| AWS provider | `>= 5.12.0` |

Source of truth: `versions.tf`. The provider constraint is intentionally a
range so each consumer can pin an exact version.

## File Layout

```
terraform-aws-cloudfront/
├── variables.tf   # 28 inputs, all optional (upstream shape)
├── main.tf        # distribution, OAC, OAI, monitoring subscription (legacy name — resources.tf is the standard)
├── outputs.tf     # distribution ids/ARNs/domain, OAI and OAC maps
├── versions.tf    # required_version + required_providers
├── wrappers/      # upstream for_each wrapper
├── examples/      # complete
├── CHANGELOG.md   # upstream changelog (ends at 3.4.0)
└── README.md      # upstream README (source line still names the public module)
```

## Inputs

| Name | Type | Default | Description |
|---|---|---|---|
| `create_distribution` | `bool` | `true` | Create the distribution. |
| `create_origin_access_identity` | `bool` | `false` | Create OAIs. |
| `origin_access_identities` | `map(string)` | `{}` | OAIs to create (value is the comment). |
| `create_origin_access_control` | `bool` | `false` | Create OACs. |
| `origin_access_control` | `map(object)` | `{ s3 = {…} }` | OACs to create, keyed by name. |
| `aliases` | `list(string)` | `null` | Alternate domain names (CNAMEs). |
| `comment` | `string` | `null` | Distribution comment. |
| `continuous_deployment_policy_id` | `string` | `null` | Continuous-deployment policy (production only). |
| `default_root_object` | `string` | `null` | Object returned for the root URL. |
| `enabled` | `bool` | `true` | Accept end-user requests. |
| `http_version` | `string` | `"http2"` | `http1.1`, `http2`, `http2and3`, or `http3`. |
| `is_ipv6_enabled` | `bool` | `null` | Enable IPv6. |
| `price_class` | `string` | `null` | `PriceClass_All`, `PriceClass_200`, or `PriceClass_100`. |
| `retain_on_delete` | `bool` | `false` | Disable instead of delete on destroy. |
| `wait_for_deployment` | `bool` | `true` | Wait for `Deployed` status. |
| `web_acl_id` | `string` | `null` | WAF web ACL (WAFv2 ARN) to attach. |
| `staging` | `bool` | `false` | Mark as a staging distribution. |
| `tags` | `map(string)` | `null` | Tags for the distribution. |
| `origin` | `any` | `null` | Origins keyed by origin id. |
| `origin_group` | `any` | `{}` | Origin groups (failover). |
| `viewer_certificate` | `any` | `{ cloudfront_default_certificate = true, … }` | TLS certificate settings. |
| `geo_restriction` | `any` | `{}` | Geo restriction. |
| `logging_config` | `any` | `{}` | Standard access-log destination. |
| `custom_error_response` | `any` | `{}` | Custom error responses. |
| `default_cache_behavior` | `any` | `null` | Default cache behavior. |
| `ordered_cache_behavior` | `any` | `[]` | Path-pattern cache behaviors, in precedence order. |
| `create_monitoring_subscription` | `bool` | `false` | Create the monitoring subscription. |
| `realtime_metrics_subscription_status` | `string` | `"Enabled"` | `Enabled` or `Disabled` real-time metrics. |

## Outputs

| Name | Description |
|---|---|
| `cloudfront_distribution_id`, `_arn`, `_domain_name`, `_hosted_zone_id` | Distribution identity and the values Route 53 aliases need. |
| `cloudfront_distribution_status`, `_etag`, `_caller_reference`, `_last_modified_time`, `_in_progress_validation_batches`, `_trusted_signers`, `_tags` | Distribution metadata. |
| `cloudfront_origin_access_identities`, `_identity_ids`, `_identity_iam_arns` | Created OAIs, for S3 bucket policies. |
| `cloudfront_origin_access_controls`, `_controls_ids` | Created OACs. |
| `cloudfront_monitoring_subscription_id` | Monitoring subscription id. |

## Examples

- `examples/complete` — distribution with S3 and custom origins, OAC, custom
  error responses, logging, and an ACM certificate.

## Validate Changes Locally

- Run `terraform fmt` before opening a PR (always — CI may fail otherwise).
- From the module dir: `terraform init -upgrade=false`.
- Run `terraform validate`.
- **Do not run `terraform plan` or `terraform apply` locally** — modules
  don't hold state on their own; that happens in the consumer stack.

## Publishing a New Version

1. Make the changes; run `terraform fmt` + `terraform validate` (and
   `terraform test` only if a `tests/` directory exists).
2. Pick the version by semver: adding a **required** input, or renaming or
   removing an output, is a **breaking change → major bump**; a new optional
   input → minor; a fix → patch.
3. Update `README.md` (and `CHANGELOG.md` if the module keeps one).
4. Tag and push **that one tag**: `git tag vX.Y.Z && git push origin vX.Y.Z`.
5. HCP Registry auto-publishes from the tag; bump consumers' `version` pins to
   the exact, `v`-stripped value (`3.1.0`, not `v3.1.0`).

## Conventions & Naming

House style and module conventions load automatically from the
`harri-tf-house-style` and `harri-tf-modules` rules whenever a `.tf` file in this
module is edited — they are not repeated here.

### Project naming patterns

- `origin` map keys are the origin ids referenced by `target_origin_id` in
  the cache behaviors; keep them stable — changing one re-creates behaviors.
- `origin_access_control` defaults to a single `s3` entry; set
  `create_origin_access_control = true` to materialise it.
- Prefer OAC (`origin_access_control`) over the legacy OAI for S3 origins.
- TerraformCloudFront pins `1.0.0`; upstream tags (`v6.x`) do not correspond to
  the Harri registry versions.
