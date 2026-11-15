---
title: "KEP-5866: Server-side Sharded List and Watch"
link: https://www.kubernetes.dev/resources/keps/5866/
shared: 2026-11-15
discussed: 2026-11-15
tags: [kubernetes, api-machinery, sharding, informers, controllers]
---

# KEP-5866: Server-side Sharded List and Watch

🔗 [Read it here](https://www.kubernetes.dev/resources/keps/5866/) · SIG API Machinery · Alpha v1.36, Beta v1.38 · feature gate `ShardedListAndWatch`

## Why read it

Adding replicas to most controllers doesn't make them faster. With leader election only one works, and with client-side sharding every replica still receives and decodes every event. This KEP lets the API server send each replica only its own share.

## Questions to think about

- Why doesn't client-side sharding reduce the load on each replica?
- Why use hash ranges instead of `hash % N`?
- Who should decide which replica owns which range?

## Takeaways

_Filled in after Sunday's chat._
