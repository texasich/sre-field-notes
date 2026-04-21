# the 504 that only happens on large uploads

small uploads work. medium uploads work. the 2gb upload your enterprise customer does once a week gets a 504 gateway timeout every time. they retry. same result. someone opens a ticket that says "uploads are broken" and your dashboard shows 99.9% success rate because the failing requests are 0.1% of volume.

---

## what's actually happening

two things in the same path, both dropping the connection, both plausibly blamed on each other.

AWS ALB has a default idle timeout of 60 seconds. if the backend takes longer than that to acknowledge the upload, ALB drops the connection and the client gets a 504. that was the first suspect and it's a real factor — but raising the ALB timeout alone did not fix it.

the second thing, which took longer to find: the internal default F5 VIP downstream of the ALB was also dropping sessions on large uploads. its session and idle timeout settings were too aggressive for multi-gigabyte traffic, and they couldn't be tuned for this use case because that VIP was shared across a large number of services. touching those settings would have affected everyone else routed through it. so we were stuck between an ALB timeout we didn't want to raise globally, and an internal VIP with timeouts we weren't allowed to raise at all.

tcpdump on the backend made it obvious eventually — the connection was going away before the app even got to respond. but the app logs and the ALB logs each pointed at the other.

---

## the pattern that fixed it

a dedicated externally signed F5 VIP for the upload path, reached via Route 53 instead of through the ALB.

before:
```
client → R53 → ALB → internal default F5 VIP (shared, aggressive timeouts) → backend
```

after:
```
client → R53 → externally signed F5 VIP (dedicated, tunable) → backend
                (api traffic still goes through the ALB path)
```

the split:

- R53 points `upload.example.com` at the new externally signed F5 VIP. `api.example.com` and everything else keeps resolving to the ALB.
- the new VIP terminates TLS with its own externally signed cert, not the ALB's ACM cert.
- because this VIP is dedicated to the upload path, its idle/connection/response timeouts could be set to 300s–600s without affecting any other service.
- the ALB is no longer in the upload path at all, so its 60s default is irrelevant for this traffic.
- the internal default VIP is no longer in the upload path either, so its shared timeout profile is irrelevant for this traffic.

two failure points removed from one flow by routing around both.

---

## why it works

the core problem wasn't "ALB timeout is too short." the core problem was that the upload path was inheriting the timeout profile of whatever shared infrastructure it happened to traverse, and none of those shared knobs could be tuned for one enterprise use case without dragging every other service along with them.

ALB's idle timeout is a blunt instrument — one knob, applied to everything. the internal default VIP was the same shape of problem one hop lower: shared, conservative defaults, owned by a platform team that (correctly) wasn't going to loosen them on one service's behalf.

splitting at DNS gives you a dedicated path with its own timeout profile. no negotiation with the platform team about shared settings. no global ALB change. one vhost, one VIP, its own cert, its own timeouts.

the externally signed cert also mattered for a boring reason: the enterprise clients doing these uploads had integrations that handled an externally signed chain more reliably than the ACM chain behind the ALB. we didn't set out to change the cert — we just got it for free by putting the new VIP in the path.

---

## takeaways

- when a 504 only happens on large uploads, assume there are at least two timeout knobs in the path and confirm both. the obvious one (ALB) is often not the only one.
- shared infrastructure with conservative defaults is a feature, not a bug. if your one endpoint needs different timeouts, don't push to loosen the shared thing — route around it.
- DNS-level traffic splitting via R53 to a dedicated backend is underrated. it lets you give one endpoint its own ops profile without touching the shared path.
- before raising any timeout, ask whether it's a timeout problem or a throughput problem. large uploads usually want multipart/chunked/resumable patterns (s3 presigned multipart, tus, etc.) before they want a longer session.
- the new dedicated VIP is a separate failure domain. it will not show up in your ALB dashboards. monitor it on its own, or you will forget it exists until it breaks at 2am.
