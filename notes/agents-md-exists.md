# read AGENTS.md before you push

a lot of OSS repos now ship an `AGENTS.md` (or `CONTRIBUTING.md` section, or repo-level policy doc) that spells out what the maintainers will and won't accept from AI-assisted contributions. read it before you open a PR. read it before you comment on one.

---

## what these files usually say

the policies vary a lot. common patterns i've seen:

- **no ai-generated PRs at all.** some projects have been burned by low-quality slop PRs and now ban the whole category. if you submit one anyway you get closed and sometimes banned.
- **disclosure required.** fine to use AI, but you have to say so in the PR description. lying about it and getting caught is worse than being upfront.
- **no AI-generated commit messages or PR descriptions.** the code can be AI-assisted, but you write the narrative yourself.
- **no AI-generated reviewer responses.** means if a maintainer asks you a question on your PR, you answer as yourself, not as a model summarizing your own PR back at them.

some combination of the above is the norm on any repo with more than a handful of regular contributors.

---

## why this matters

maintainers can tell. the tells are not subtle — overconfident prose, every bullet a complete thought, em-dashes everywhere, mid-sentence "not X, but Y" constructions, unprompted "Summary" headers on two-line changes. if you ghost-write and get caught you lose credibility on that repo forever, and the OSS world is small.

disclosing honestly costs you nothing. most maintainers don't mind AI-assisted contributions if the contributor has actually read the code, tested the change, and stands behind it.

---

## what to actually do

1. before your first PR on a new repo, grep for `AGENTS.md`, check `CONTRIBUTING.md`, skim the repo's community docs.
2. if there's a policy, follow it. if you disagree with it, don't contribute — or raise it with maintainers, don't just ignore it.
3. if you're using AI assistance, disclose it in the PR body. one sentence is enough.
4. read your own PR description out loud before submitting. if it sounds like a model, rewrite it in your own voice.

the baseline here is respect for the people running the project. their rules, their house.
