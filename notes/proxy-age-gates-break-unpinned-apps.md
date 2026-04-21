# proxy age gates break everything you forgot to pin

security controls that assume everyone pins their versions will punish every team that doesn't.

---

## the setup

implemented a proxy-level restriction in nexus iq firewall → nexus repository: block any package in the proxy younger than 10 days. the threat model was solid. new packages are the highest-risk window for supply chain attacks — typosquatting, dependency confusion, compromised-maintainer takeovers, malicious versions published after an account hijack. most of that gets caught or yanked in the first few days. so: block anything published less than 10 days ago, let the community vet it, then let it through.

on paper this is good. in practice it assumes something that wasn't true.

---

## what actually happened

it broke a lot of internal apps.

turned out most teams weren't pinning. builds were pulling `latest`, caret ranges, `^x.y.z`, floating majors, whatever. the moment a popular upstream library cut a legitimate new patch release, every build that resolved to it hit the 10-day wall and failed. nothing malicious. just a new version of a package somebody depended on loosely.

one upstream release could turn 50 builds red across the org, and the failure mode was opaque — teams weren't getting "blocked by age gate," they were getting resolution errors that looked like network flakes or registry issues. triage burned hours per team before anyone connected it back to the new policy.

---

## the actual lesson

this is "pin your hashes" applied at organizational scale. an individual developer can pin. fine. but when you enforce it at the proxy level for 5,000 apps, the blast radius of every team that didn't pin becomes *your* problem to own and explain.

the correct order of operations:

1. audit pinning compliance across the org first. know which apps float vs pin.
2. enforce pinning through policy — linting, CI checks, PR gates — before touching the proxy.
3. verify compliance is actually high, not just "we sent an email about it."
4. *then* add the age gate.

we did it in the wrong order. the proxy rule was the forcing function for pinning, which meant every non-compliant team learned about the policy by having their build break at an inconvenient time.

---

## practical takeaways

- audit pinning compliance before you deploy proxy restrictions. you need the data before the policy, not after.
- roll out age gates gradually. start in monitoring/alerting mode — log what *would* have been blocked, review it, size the blast radius, *then* flip to blocking.
- provide an escape hatch during transition. package-level or team-level allowlist for the legitimate exceptions you'll discover in week one.
- the best proxy policy is one that matches the org's actual maturity level, not the one you wish they had. ship the policy your fleet can survive, not the policy the threat model wants.

---

## short version

you can't deploy a proxy-level age gate without first auditing which apps are pinning vs floating. enforce pinning first, verify compliance, then add the gate. doing it the other way around means the teams who didn't pin find out by having their builds go red, and the security win is paid for in lost trust from every team you surprised.
