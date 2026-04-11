# terraform apply is roulette

at some point, every team that uses terraform encounters the same temptation: automate the apply. just have CI run `terraform apply` after a merge. or better yet, let the AI assistant handle it — it knows the codebase, it can read the plan, it can just... apply.

no.

---

## the plan review is the whole point

the entire value proposition of terraform's plan/apply separation is that a human looks at what's about to happen before it happens. `terraform plan` is not a formality before the real work. it is the real work.

this matters because terraform's diff can be surprising in ways that aren't obvious from the code change:

- a rename in your config can show as a destroy + create on the resource (which might involve downtime, data loss, or a change in resource ID that breaks downstream references)
- a change in a provider version can alter behavior on resources you didn't touch
- `aws_db_instance` with `apply_immediately = false` vs `true` has very different implications for when that change actually happens
- `lifecycle { prevent_destroy = true }` is a lint check, not a lock — someone can remove it and apply in the same PR

the plan tells you what's actually going to happen. the diff tells you what changed in code. these are not the same thing.

---

## humans AND AIs should not apply unsupervised

this is worth stating directly: letting an AI assistant run `terraform apply` on production is the same category of mistake as letting it run unsupervised in general. the model doesn't have production context. it doesn't know that the "small" rds modification requires a multi-hour maintenance window. it doesn't know that this particular module has a bug where every apply rotates the KMS key.

the model can help you read the plan. it can flag things that look wrong. it can draft the apply pipeline config. it should not have unilateral apply authority on anything that touches real infra.

same thing applies to humans, but we've been saying that for years and people keep ignoring it.

---

## what a reasonable apply process looks like

**local/dev:**
- `terraform plan` output reviewed locally
- `terraform apply` on a personal sandbox is fine without ceremony

**staging:**
- plan generated in CI, output stored as artifact
- apply triggered manually (button click / comment / whatever), not automatic
- plan artifact is what gets applied, not a fresh plan at apply time (eliminates TOCTOU drift)

**production:**
- same as staging, plus
- plan reviewed by at least one person who didn't write the change
- applied during a known maintenance window or change freeze window if it touches stateful resources
- rollback path documented before apply starts (for terraform this often means knowing the blast radius, not just running `terraform destroy`)

---

## the gotcha that bites everyone once

`terraform apply` on a fresh plan vs a saved plan.

if your CI runs `terraform plan` and posts the output to the PR, and then runs `terraform apply` (without `-plan=`) when you merge — it's computing a new plan at merge time. whatever infrastructure drift or state changes happened between the PR being posted and the merge can change the apply output. you approved a plan that's no longer what's being applied.

the fix: save the plan with `terraform plan -out=tfplan` and then `terraform apply tfplan`. what you reviewed is exactly what gets applied.

---

## short version

read the plan. every time. for real. the 10 minutes you spend on plan review is buying you out of incidents that take 10 hours to recover from.
