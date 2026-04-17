# when in doubt, ship the revert

your merged PR regressed something in public. maybe the build is red, maybe a test is flaking, maybe a feature quietly got slower. you have two options:

1. land a follow-up fix that addresses the root cause.
2. revert your PR, then come back later with a fix.

option 2 is almost always the right call. and almost nobody's instinct picks it.

---

## why the revert wins

the cost of a red master is not linear. every hour master is broken:

- every contributor who pulls has a broken checkout
- every CI run wastes compute on failures that aren't their fault
- maintainers burn cycles triaging noise
- your name is on the regression the whole time

the cost of a revert is:

- one extra commit in the log
- you take another crack later, with no time pressure, with a proper test plan

the math is not close. reputation-wise, "shipped a regression, reverted fast, fixed it properly next week" is how competent engineers operate. "shipped a regression, spent four hours forward-fixing under pressure, broke something else" is how you become the cautionary tale.

---

## when to actually forward-fix

there are cases. if the fix is obviously one line and you can prove it locally in under 10 minutes, forward-fix. if the revert itself would cause worse breakage (because other things landed on top of it), forward-fix carefully.

everything else: revert first, be the grown-up, come back when your hands aren't shaking.

the ego hit of "i have to revert my own PR" is the tax. pay it.
