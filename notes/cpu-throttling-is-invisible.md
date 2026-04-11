# cpu throttling is invisible

your dashboard is green. p99 latency is... fine-ish. cpu utilization shows 40%. nothing is alerting. but your service is slow and nobody can explain why.

nine times out of ten in a kubernetes environment: cpu throttling.

---

## how cpu limits actually work

kubernetes cpu limits aren't a speed cap in the intuitive sense. they're enforced by the linux CFS (completely fair scheduler) using quota/period windows. by default, kubernetes uses 100ms periods. if your container's limit is 500m (0.5 cores), it gets 50ms of cpu time per 100ms window.

the catch: even if the node has plenty of idle cpu, your container gets throttled when it hits the quota for that window. it sits and waits for the next period. no error. no OOMKill. just... waiting.

from the outside this looks like normal latency variance or a slow dependency. you'll spend an hour checking your database before someone pulls up the right prometheus metric.

---

## the metric you're not watching

```
container_cpu_cfs_throttled_seconds_total
```

or in a more useful form:

```
rate(container_cpu_cfs_throttled_seconds_total[5m])
  / rate(container_cpu_cfs_periods_total[5m])
```

that gives you throttle ratio — what percentage of scheduling periods your container is being throttled. anything above 25% is worth investigating. i've seen teams sitting at 70-80% throttle ratio on their critical path services wondering why p99 was spiking.

the reason nobody catches it: cpu utilization charts show average usage. throttling happens in bursts within scheduling windows. you can have 40% average utilization and 60% throttle ratio at the same time.

---

## memory limits vs cpu limits: very different failure modes

this is worth understanding clearly:

- **memory limit hit → OOMKill.** immediate. loud. your pod restarts, you get a CrashLoopBackOff, prometheus fires, someone notices.
- **cpu limit hit → throttling.** silent. your pod keeps running. nothing crashes. the scheduler just slows your container down.

memory limits give you fast, visible failures. cpu limits give you slow, hidden degradation. both are bad but in completely different ways. the visibility asymmetry is the issue.

---

## what to do about it

first, instrument it. add `container_cpu_cfs_throttled_seconds_total` to your dashboards and put an alert on throttle ratio > 25% for anything in your critical path. you'll probably find a few surprises the first time you do this.

second, think carefully before setting cpu limits at all. there's a legitimate school of thought (see: some of the kubernetes performance tuning literature) that says don't set cpu limits, only set requests. let the scheduler use requests for bin-packing and let containers burst freely when cpu is available. the downside is noisy neighbor risk; the upside is you don't silently throttle well-behaved workloads.

if you do set limits, set them with headroom. if your average cpu is 200m, don't set your limit at 250m. set it at 1000m and let it burst. the request is what the scheduler uses for placement; the limit is the ceiling. they don't have to be close together.

third, if you're running java or jvm-based services, watch out for gc pause interaction with throttling. gc already causes latency spikes; throttle a jvm during gc and you'll have a bad time.

---

## the actual takeaway

add `container_cpu_cfs_throttled_seconds_total` to your dashboards today. before you hit a slow incident and spend three hours staring at the wrong metrics.
