# sre-field-notes

14 years of shipping things and being on-call when they break. this is where i keep the notes.

not a tutorial series. not a certification guide. no "10x your k8s game" energy here. just opinionated field notes from real production — the kind of stuff i wish someone had written down before i learned it the hard way at 2am.

if your pipeline's held together by one guy named dave, i've been dave.

---

**disclaimer:** these are opinions from experience. they worked in the contexts i've been in. your infrastructure is different. treat everything here as a starting point, not a runbook you paste into prod without thinking.

twitter: [@quant_papi](https://twitter.com/quant_papi)

---

## notes

**kubernetes**
- [kubectl auth at 3am](notes/kubectl-auth-at-3am.md) — service account token expiry, OIDC vs x509 at scale
- [cpu throttling is invisible](notes/cpu-throttling-is-invisible.md) — why your dashboard shows 40% cpu while your service is slow
- [priorityclasses and topology spread](notes/priorityclasses-and-topology-spread.md) — eviction isn't random if you configure it, and chaos testing is not optional
- [headless services for stateful workloads](notes/headless-services-for-stateful.md) — clusterIP: None and why kafka doesn't belong behind a load balancer
- [kind → k3s → eks](notes/kind-to-k3s-to-eks.md) — the progression for actually understanding kubernetes before AWS hides it from you

**terraform / aws**
- [terraform apply is roulette](notes/terraform-apply-is-roulette.md) — the plan review is the whole point; don't automate past it
- [copy_tags_to_snapshot](notes/copy-tags-to-snapshot.md) — the one-liner that keeps your RDS snapshots from failing compliance

**supply chain**
- [pin your hashes](notes/pin-your-hashes.md) — unpinned installs on boxes with credentials are a pending postmortem
