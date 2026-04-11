# headless services for stateful workloads

a kubernetes Service normally gives you a stable virtual IP (ClusterIP) and load balances traffic across the matching pods. that's the right model for stateless applications where any pod can handle any request.

stateful workloads are different. kafka brokers aren't interchangeable. database primaries and replicas aren't interchangeable. cassandra nodes have identity — the cluster topology depends on which node is which. putting a load balancer in front of these things ranges from useless to actively harmful.

headless services are the answer.

---

## what a headless service actually does

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
spec:
  clusterIP: None   # this is the headless part
  selector:
    app: kafka
  ports:
    - port: 9092
      name: client
```

`clusterIP: None` tells kubernetes: don't allocate a virtual IP, don't set up load balancing. instead, DNS queries for `kafka.namespace.svc.cluster.local` return A records for each matching pod directly.

more importantly, each pod in a StatefulSet gets its own stable DNS entry:
- `kafka-0.kafka.namespace.svc.cluster.local`
- `kafka-1.kafka.namespace.svc.cluster.local`
- `kafka-2.kafka.namespace.svc.cluster.local`

these DNS names persist across pod restarts (as long as the StatefulSet ordinal stays the same). the pod gets a new IP when it restarts, but the DNS name is stable. consumers and brokers can reconnect by name.

---

## why this matters for kafka specifically

kafka brokers advertise their address to clients. the client connects to a bootstrap broker, gets the full broker list, and then connects directly to individual brokers for partition reads/writes.

if your kafka brokers are behind a regular ClusterIP service, the advertised address is the load balancer IP. every direct broker connection goes through the lb. this breaks partition-aware clients and can cause significant overhead.

with a headless service + StatefulSet:
- each broker advertises its own stable pod DNS name
- clients connect directly to the right broker for each partition
- no unnecessary load balancer round-trips on the critical path

same pattern applies to zookeeper, cassandra, scylladb, postgres with patroni — anything where the client needs to know about individual node identity.

---

## the StatefulSet requirement

headless services only give you the stable pod DNS if you're using a StatefulSet. a Deployment doesn't guarantee pod names or ordinals — pods get random suffixes and if you scale down and back up you might get different names.

StatefulSet guarantees:
- ordered, stable pod names (0, 1, 2...)
- ordered startup and shutdown (pod N-1 before pod N)
- stable network identity (pod name = DNS subdomain)
- stable persistent volume claims (PVC follows the pod across rescheduling)

the ordered startup matters for things like kafka and zookeeper where you don't want to bring the whole cluster up simultaneously and have them fight over leadership.

---

## debugging tip

to verify the headless DNS is working from inside the cluster:

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# inside the container:
nslookup kafka.namespace.svc.cluster.local
```

a headless service returns multiple A records (one per pod). a regular ClusterIP service returns one A record for the virtual IP. if you're getting one IP, something's wrong with your `clusterIP: None`.

---

## short version

use `clusterIP: None` (headless services) + StatefulSets for anything where pod identity matters. it gives you stable per-pod DNS names, direct pod-to-pod connectivity, and the right primitives for stateful distributed systems. don't put kafka behind a regular ClusterIP service.
