# copy_tags_to_snapshot

small one. bit me on a compliance audit. adding it here so someone else doesn't have to learn it this way.

---

## the gotcha

you've got an RDS instance managed with terraform. it has tags: `Environment = production`, `ManagedBy = terraform`, `CostCenter = platform`, whatever your tagging standard is.

those tags live on the instance. when RDS takes automated snapshots (daily backups, or whatever retention window you've configured), the snapshots do not inherit those tags by default.

so now you have:
- RDS instance: properly tagged, shows up in cost reports, passes your tagging compliance check
- automated snapshots of that instance: untagged, orphaned in your tagging policy, invisible to your cost allocation, may fail security scans that require tags on all resources

if your organization has a "all resources must be tagged" policy enforced by AWS Config or a similar tool, your snapshots are failing that check silently. depending on how strict your setup is, automated snapshots in a disaster recovery scenario might also be harder to identify and attribute if they're not tagged.

---

## the fix

```hcl
resource "aws_db_instance" "main" {
  # ... your other config ...

  copy_tags_to_snapshot = true
}
```

one line. that's it. now automated snapshots inherit the instance tags at creation time.

the frustrating thing is that this defaults to `false`. it's opt-in behavior that you have to know to look for. it's not surfaced prominently in the console or in most getting-started guides.

---

## the broader pattern

there are a handful of AWS resource config options that follow this same pattern: sensible-seeming defaults that cause quiet compliance or operational problems downstream. some others i've hit:

- **RDS deletion protection**: defaults to `false`. `deletion_protection = true` should be in every production module template.
- **RDS auto minor version upgrade**: defaults to `true`, which means AWS can auto-upgrade your minor version during a maintenance window. sometimes that's fine. sometimes it's a surprise. make it explicit.
- **S3 bucket versioning**: off by default. if you're using a bucket as a tfstate backend or storing anything you might need to recover, `versioning { enabled = true }` should be your default.
- **CloudWatch log retention**: defaults to never expire. your logs will grow forever and you'll pay for them forever. set a retention policy.

none of these are gotchas in the sense of being undocumented. they're all in the AWS docs. the issue is that the defaults are whatever made sense for general use, and "general use" isn't "production infrastructure that needs to meet compliance requirements."

---

## short version

add `copy_tags_to_snapshot = true` to your `aws_db_instance` resources. and audit your other RDS defaults while you're in there.
