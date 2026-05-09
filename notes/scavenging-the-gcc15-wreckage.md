# scavenging the gcc15 wreckage

gcc 15 shipped with `-std=gnu23` as the default. that one flag broke hundreds of c and c++ projects overnight. `bool` is now a keyword. `u_char` disappeared from sys/types.h. `bsearch()` returns `const void*` in glibc 2.43. typedefs that worked for 30 years became compile errors.

the easy bugs got scooped within a month. by the time i went looking in may 2026, every single target from my original plan was closed. jq. iperf. nethogs. rsyslog. e2fsprogs. all of them. the low-hanging fruit was gone.

so i learned to scavenge.

---

## the real skill is finding unclaimed bugs

writing the fix is usually one line. the hard part is finding a bug that hasn't already been claimed by three other people.

i almost opened a PR for rocksdb — missing `<cstdint>` includes in their blob metadata headers, breaking gcc 15 builds. checked the repo first. four open PRs for the same bug. oldest one was 15 months old. meta's maintainer pipeline is a parking lot and i would've been car number five.

same thing with vllm. gemma3's mmproj gguf file not getting downloaded alongside model weights. looked clean — zero comments on the issue, two months old. checked for existing PRs. two already open. vllm explicitly bans duplicate PRs in their AGENTS.md. saved myself a wasted afternoon.

the lesson: `gh pr list` and `gh issue view --comments` before you write a single line. the first person to find the bug is rarely the first person to open a PR.

---

## what actually shipped

**strace — one cast, massive surface area**

glibc 2.43 changed `bsearch()` to return `const void *` under c23. strace's `ioctl_lookup()` assigns the result to `const struct_ioctlent *` without an explicit cast. gcc 15 says no. the fix is one `(const struct_ioctlent *)` cast. strace runs on every linux box on the planet. one line, massive impact.

opened [PR #394](https://github.com/strace/strace/pull/394). zero competition. sometimes the best bugs are the ones that look too obvious to still be open.

**rpcemu-extended — bool is a keyword now**

`typedef int bool;` in a c file. gcc 15 says bool is reserved. replaced with `#include <stdbool.h>`. [PR #15](https://github.com/andrewtimmins/rpcemu-extended/pull/15). tiny project, but the fix is archetypal — this exact pattern broke dozens of repos.

**libpcap — filing upstream to unblock downstream**

nethogs maintainer said "i'll merge this workaround, but can you file an upstream issue at libpcap?" so i did. [libpcap #1679](https://github.com/the-tcpdump-group/libpcap/issues/1679) — requesting they add `u_char`/`u_short`/`u_int` compat typedefs in their public headers. linked it in the nethogs PR comment. maintainer said thanks. that's how you unblock a merge without writing more code.

---

## the agent workflow that actually works

i've been running this with hermes orchestrating, deepseek v4pro as the primary coder, and gemini for research and vision. the loop:

1. scout with `gh search issues` — broad queries, filter out noise, verify nothing is claimed
2. clone fresh, apply the fix, verify with a syntax check
3. push to fork, open PR, update the plan tracker

the agent doesn't need to understand the whole codebase. it needs to find a bug, verify the fix compiles, and not step on anyone else's work. the diligence is the product.

what killed time was chasing dead ends. griddb had an i386-specific ftbfs that needed autoconf changes and couldn't be tested on an x86_64 droplet. vllm had the duplicate PR problem. rocksdb had four competitors. the scouting-to-shipping ratio was maybe 5:1 — five bugs investigated for every one that shipped.

---

## short version

gcc 15 created a scavenger hunt across the entire c ecosystem. the easy stuff is gone. what's left takes more digging than coding. always check for existing PRs before you touch a file. sometimes the best contribution is filing an upstream issue that unblocks someone else's merge.

three shipped, two dead ends, one lesson: you are never the first person to find the bug.
