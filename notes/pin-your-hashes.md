# pin your hashes

`pip install requests` is not a deployment strategy. it's a pending postmortem.

---

## the threat model

supply chain attacks on open source packages are not hypothetical. they have a specific pattern: attacker publishes a malicious version of a popular package (typosquatting, compromised maintainer account, malicious PR merged during maintainer inactivity), package gets pulled in by automated installs with unpinned or loosely-pinned dependencies, malicious code runs on any machine with access to credentials — which, in a CI/CD pipeline, is often everything.

the attack surface is large because:
1. most teams have long dependency chains. your direct dependencies have their own dependencies. those have dependencies.
2. build environments often have implicit access to production credentials, deploy keys, or AWS instance roles.
3. unpinned installs are lazy-evaluated — what was safe last week might not be safe today.

---

## what pinning actually means

**not this:**
```
# requirements.txt
requests>=2.28.0
boto3
numpy
```

**this:**
```
# requirements.txt
requests==2.31.0 --hash=sha256:58cd2187423d...
boto3==1.34.11 --hash=sha256:7a2d9e9c4b...
```

the hash is the part that matters. `requests==2.31.0` pins the version but a compromised registry could serve a different file at that version. the hash pins the exact artifact. if someone serves you something else, pip refuses to install it.

same principle applies everywhere:
- **npm/yarn**: `package-lock.json` with integrity hashes. commit it. `npm ci` instead of `npm install` in CI (respects lockfile exactly, errors on drift).
- **go**: `go.sum` is hashes by default. don't delete it, don't gitignore it.
- **docker**: `FROM python:3.11` is "whatever was 3.11 when this runs." `FROM python:3.11@sha256:abc123...` is a specific image forever.
- **github actions**: `uses: actions/checkout@v3` resolves to whatever the maintainer points that tag at. `uses: actions/checkout@sha256:abc123` does not.

---

## the CI/CD angle

your pipeline box probably has:
- aws credentials / instance role
- github deploy keys or tokens
- npm publish credentials
- container registry push access

if your build step does `pip install -r requirements.txt` without hash verification on a compromised package, the malicious code runs in that environment with all of those credentials available.

i've seen this exact scenario play out. not in a headline-making way — just a team that had loose pins, a compromised transitive dependency, and credentials leaked to an unknown endpoint before anyone noticed something was wrong. the detection was months later during a security audit.

---

## making it less painful

the maintenance burden of pinning is real. it means regular updates and hash regeneration. tools that help:

- **pip-tools** (`pip-compile`): generates fully-pinned `requirements.txt` with hashes from a `requirements.in` spec
- **dependabot / renovate**: automated PRs when dependencies have updates. review them, don't just merge.
- **docker scout / grype / trivy**: scan your images for known CVEs. run in CI.

the goal isn't to never update. it's to make updates explicit and reviewed, not implicit and automatic.

---

## short version

every `pip install` / `npm install` / `docker pull` without pinning on a machine with credentials is a risk you're accepting. pin hashes in your lockfiles, commit the lockfiles, use `ci` install modes in pipelines, and treat dep updates as code changes that get reviewed.
