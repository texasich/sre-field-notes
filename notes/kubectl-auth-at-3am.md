# kubectl auth at 3am

you get the page. cluster's healthy by every metric that matters — nodes up, pods running, no obvious OOMKills. but one service is throwing 401s and your on-call engineer can't `kubectl exec` into anything. no errors in the app logs. just silence and 401s.

forty minutes in, someone figures it out: the service account token expired.

---

## the actual problem

kubernetes service account tokens don't last forever. the default token expiration behavior has changed across versions — in older clusters, the tokens were technically non-expiring by default but that changed in 1.24 when bound service account tokens became the norm with a 1-hour default TTL (configurable, but often left at default).

if you rotated a token or the token hit its TTL and nothing auto-refreshed it, your pods are now running with stale credentials. the auth failure is silent unless you're specifically watching for it.

the deeper problem: most teams don't watch for it. they watch for cpu, memory, error rates. not token TTL.

---

## what actually works at scale

**OIDC with your cloud IAM is the answer.** 

for AWS: IAM Roles for Service Accounts (IRSA). pod gets an annotation pointing to an IAM role, the kubelet injects a projected volume with a short-lived OIDC token, aws-sdk picks it up automatically, token auto-refreshes on a ~1 hour cycle. you never touch a static credential again.

for GKE: workload identity does the same thing. for AKS: workload identity federation.

the pattern is the same everywhere: let the platform handle token lifecycle. stop managing static credentials in secrets and mounting them as env vars. that model doesn't scale and it will wake you up eventually.

---

## x509 client certs: fine at small scale, painful at large

x509 certs are what kubeconfig uses by default for user auth. totally fine if you have 5 engineers and one cluster. at 50 engineers and 20 clusters it's a coordination nightmare — you need a PKI, you need rotation automation, you need revocation (which kubernetes doesn't actually support for certs the way you'd expect, since there's no native CRL check in the api server by default).

i've seen teams get burned by this: an engineer leaves, their cert is still valid for 364 more days, nobody revoked it because there was no revocation process, and now you have an audit finding.

OIDC + your IdP (okta, google workspace, whatever you use) solves this: user offboarded from IdP = no more cluster access. clean.

---

## what to actually do

1. if you're starting fresh: set up OIDC auth from day one. non-negotiable.
2. for pod auth: IRSA / workload identity. delete your static IAM keys from secrets.
3. audit your cluster's token expiration settings. check `--service-account-token-volume-projection` and `--service-account-max-token-expiration` on your api server flags.
4. add a check in your service's health logic or startup probe that validates auth against the downstream dependency, not just "can i reach the endpoint."

the 401 at 3am is not the problem. the problem is you didn't know the token was going to expire.
