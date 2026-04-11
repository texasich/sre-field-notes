# priorityclasses and topology spread

you've got 5 replicas of your api service. the cluster gets tight on resources. kubernetes starts evicting pods. later you find out it evicted 4 of your 5 api replicas and kept the batch job that runs once a day. or: kubernetes spread your 5 replicas "across nodes" and they all landed on the same AZ, and then that AZ had a network hiccup.

both of these are preventable. most teams don't configure for them until after the incident.

---

## priorityclasses: eviction isn't random

when the node runs low on memory, the kubelet has to evict something. how does it pick? it's a combination of QoS class (based on whether you've set requests/limits) and PriorityClass.

if you haven't set PriorityClasses, your api service and your nightly batch job look roughly equivalent to the scheduler. it'll make a decision you won't like.

the fix is simple:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-services
value: 1000000
globalDefault: false
description: "production-facing services that should survive eviction pressure"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-workloads
value: 10000
globalDefault: false
description: "batch jobs, non-critical background work"
```

then set `priorityClassName: critical-services` on your production deployments and `priorityClassName: batch-workloads` on your cron jobs and background workers. now when the cluster is under pressure, the scheduler knows what to keep.

kubernetes ships with `system-cluster-critical` and `system-node-critical` built in (values 2000000000 and 2000001000). your workloads should sit below those. pick a scheme that maps to your actual service tiers and stick with it.

---

## topologySpreadConstraints: don't put all your eggs in one node

the default pod anti-affinity is tempting but blunt. topologySpreadConstraints lets you express what you actually want: spread evenly across zones, with a max skew you can tolerate.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-api
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: my-api
```

this says: try to keep zone distribution within 1 pod of even (hard constraint), and try to spread across nodes (soft constraint, don't block scheduling if you can't). adjust `whenUnsatisfiable` based on whether you'd rather have 4/5 pods running or 0/5 waiting for perfect placement.

the `kubernetes.io/hostname` spread is the one people miss. you can have 3 AZs with perfect zone balance and still have 4 replicas on one physical node if you don't also spread by hostname.

---

## actually test node failure

the above is only useful if you've validated it. configuration that's never been exercised is just a theory.

tools that make this testable:

- **chaos-mesh**: inject node failures, network partitions, pod kills on a schedule. runs in-cluster. good for CI integration.
- **litmus chaos**: similar, more opinionated hub model, good CNCF project.

the workflow: deploy your chaos experiment in a test environment, define a steady-state hypothesis (e.g., "p99 latency stays under 500ms"), trigger a node failure, assert the hypothesis holds. if it doesn't, fix your priorityclasses and topology config and try again.

this sounds like extra work until you're explaining a 45-minute outage because kubernetes evicted the wrong pods during a memory spike on a tuesday afternoon.

---

## short version

- set PriorityClasses before your cluster ever hits resource pressure, not after
- use topologySpreadConstraints for zone + hostname spread on anything with > 1 replica
- test node failures with chaos-mesh or litmus in a non-prod environment at least once a quarter
