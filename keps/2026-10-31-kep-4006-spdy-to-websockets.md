---
title: "KEP-4006: Transition from SPDY to WebSockets"
link: https://www.kubernetes.dev/resources/keps/4006/
shared: 2026-10-31
discussed: 2026-11-01
tags: [kubernetes, api-machinery, sig-cli, kubectl, networking]
---

# KEP-4006: Transition from SPDY to WebSockets

🔗 [Read it here](https://www.kubernetes.dev/resources/keps/4006/)

## Why read it

Every `kubectl exec`, `attach`, `cp` and `port-forward` has been riding on SPDY, a protocol deprecated since 2015 that many proxies and load balancers no longer support. This KEP moves that streaming to WebSockets, and shows how you change a protocol under a huge user base without breaking old clients.

## Questions to think about

- Why did SPDY become a problem for `kubectl` behind proxies, gateways and load balancers?
- How does the API server stay compatible with old clients and kubelets that still speak SPDY?
- What did WebSockets need on top (extra subprotocol versions, a close signal) to replace SPDY's multiple streams?

## Takeaways

_Filled in after Sunday's chat._
