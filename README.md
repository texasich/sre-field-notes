# sre-field-notes

14 years of shipping things and being on-call when they break. this is where i keep the notes.

not a tutorial series. not a certification guide. no "10x your k8s game" energy here. just opinionated field notes from real production — the kind of stuff i wish someone had written down before i had to learn it the hard way at 2am.

if your pipeline's held together by one guy named dave, i've been dave.

---

**disclaimer:** these are opinions from experience. they worked in the contexts i've been in. your infrastructure is different. treat everything here as a starting point, not a runbook you paste into prod without thinking.

twitter: [@quant_papi](https://twitter.com/quant_papi)

---

## notes

- [kubectl auth at 3am](notes/kubectl-auth-at-3am.md)
- [cpu throttling is invisible](notes/cpu-throttling-is-invisible.md)
- [priorityclasses and topology spread](notes/priorityclasses-and-topology-spread.md)
- [terraform apply is roulette](notes/terraform-apply-is-roulette.md)
- [kind → k3s → eks](notes/kind-to-k3s-to-eks.md)
- [pin your hashes](notes/pin-your-hashes.md)
- [headless services for stateful workloads](notes/headless-services-for-stateful.md)
- [copy_tags_to_snapshot](notes/copy-tags-to-snapshot.md)
