# kind → k3s → eks

the most common mistake i see people make when learning kubernetes: they start with EKS or GKE.

they get a managed cluster, kubectl starts working, they deploy a hello-world, and they think they understand kubernetes. they don't. they understand how to use a managed control plane. when something breaks, they have no model for why.

---

## the progression that actually works

**step 1: kind (kubernetes in docker)**

kind runs a full kubernetes cluster locally inside docker containers. it's not a toy — it's what a lot of k8s upstream tests run on. you get real multi-node topology (one control plane, multiple workers, all docker containers), real CNI, real everything.

start here because:
- fast iteration. `kind create cluster` takes ~30 seconds. break something? delete and recreate.
- no cost. break as many clusters as you want.
- you learn the shape of the system before the shape of someone else's managed version of it

spend time actually breaking things. delete a node container. watch what happens to pods. mess with resource limits until you get OOMKills. read the kubelet logs. understand what the api server actually does.

**step 2: k3s on a $5 vps**

kind hides real networking. you don't learn anything about actual node-to-pod routing, CNI plugins in a real environment, or what happens when your network has real latency and packet loss.

spin up a cheap VPS (hetzner, linode, wherever) and install k3s. then install it again with a second node and try to get them to actually join. you'll hit:

- firewall rules blocking cluster traffic you didn't know existed
- flannel vs calico choices that actually matter
- etcd vs sqlite for small clusters
- kubeconfig handling when you have multiple contexts

the $5 box teaches you what managed services abstract away. also: debug a broken k3s node without vendor support. it changes how you think about the system.

**step 3: EKS / GKE / AKS**

now use the managed service. at this point you understand:
- what the managed control plane is actually doing for you (and what it isn't)
- why node IAM roles and IRSA exist and what problem they solve
- why managed node groups have opinions about AMI and launch templates
- what "bring your own CNI" actually means on EKS

you can read the docs and understand them instead of just copying them. when something breaks in production you have a mental model, not just a support ticket.

---

## what skipping ahead costs you

when teams start with EKS:

- they think kubernetes networking works the way the aws docs describe it (it doesn't, really — the docs describe the AWS layer on top)
- they can't debug pod networking issues because they've never had to
- they're dependent on whoever set up the cluster originally because none of the internals make sense
- when the cluster has a problem that isn't answered by the managed service docs, they're stuck

i've interviewed plenty of people with "3 years kubernetes experience" on their resume who've only ever used GKE and have never debugged a CNI issue, a scheduler problem, or a kubelet failure directly. that experience is real — but it has gaps that matter when things go wrong.

---

## the caveat

you don't have to master bare-metal kubernetes to use EKS well. the goal isn't "run your own cluster in production." the goal is: understand the system well enough that managed services are a productivity tool, not a mystery box you're hoping stays healthy.

kind → k3s → managed is the fastest path to that. EKS first is paying AWS to hide the learning.
