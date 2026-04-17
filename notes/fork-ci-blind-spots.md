# your fork ci can't see everything

"all green on my fork" is not the same as "all green on the upstream PR." if you've been working mostly on internal repos this catches you off guard.

---

## why forks don't run the full matrix

most serious OSS projects have a CI matrix that includes:

- self-hosted runners (arm boxes, gpu boxes, windows-on-arm, embedded targets)
- jobs gated on secrets the fork doesn't have access to (signing keys, deploy creds, api tokens)
- jobs that only trigger on `pull_request_target` with maintainer approval

none of these run on your fork. github's default behavior is that forks get the github-hosted runners for free and nothing else. this is correct from a security standpoint — you don't want every drive-by fork to be able to exercise your signing key — but it means your fork's green checkmark is a subset of what the maintainers will see.

---

## how this bites you

you push a build-system change. fork CI: green across linux, mac, windows hosted runners. you open the PR. the upstream runs its matrix including a self-hosted raspberry pi build that catches an ARM-specific regression. now you're the person with the red PR on a weekday morning.

specifically dangerous for: cmake changes, toolchain pins, compiler flag changes, anything that touches cross-compilation. these are the exact classes of change where the hidden jobs are most likely to fail.

---

## what to do

1. after pushing, **actually look at the upstream PR checks tab**. not your fork's actions tab. they are different pages with different jobs.
2. if you're touching anything build-system-ish, ask a maintainer to kick off the full matrix before you assume you're done.
3. if the project has a `CONTRIBUTING.md` or equivalent, read what it says about the full CI matrix and whether fork runs are trusted.
4. budget time for a round-trip where the upstream finds something your fork couldn't.

the fork's green check is necessary, not sufficient.
