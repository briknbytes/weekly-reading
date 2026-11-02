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

**TL;DR:** `kubectl exec`, `attach`, `cp` and `port-forward` moved from SPDY (deprecated since 2015) to WebSockets so they work behind modern proxies and load balancers. GA in v1.38.

- **Why:** proxies and load balancers dropped SPDY, which broke streaming. HTTP/2 wasn't an option because it forbids `Upgrade`.
- **How it stays compatible:** kubectl tries WebSockets and falls back to SPDY. Phase 1: the API server translates WebSocket to SPDY. Phase 2: kubelets that advertise support take the WebSocket directly. The kubelet → runtime hop stays SPDY.
- **What WebSockets needed:** a one-byte channel prefix and a new `v5.channel.k8s.io` subprotocol with a CLOSE signal, so stdin can end without hanging.
- **Security twist:** the WebSocket handshake must be a GET, which maps to the RBAC verb `get`. "Read-only" roles could open a shell. Fix: an extra `create` check on `pods/exec`, `attach` and `portforward`, on by default since v1.35 and locked on in v1.38.
- **Seen live:** `kubectl exec -v=8` showed `GET` + `Upgrade: websocket` + `v5.channel.k8s.io` + `101`. Forcing SPDY showed `POST` + `Upgrade: SPDY/3.1`.

**Under the hood:**
- **kubectl** tries WebSockets first and falls back to SPDY → [`remotecommand/fallback.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/tools/remotecommand/fallback.go)
- **Channel bytes** (0 stdin, 1 stdout, 2 stderr, 3 error, 4 resize, 255 close) → [`remotecommand/constants.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/util/remotecommand/constants.go)
- **API server / kubelet** translate WebSocket → SPDY → [`proxy/streamtranslator.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apiserver/pkg/util/proxy/streamtranslator.go)
- **kubelet** asks the runtime for a streaming URL and proxies to it → [`kubelet/server/server.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/server/server.go) (`getExec`)
- **CRI contract:** `Exec` returns only a URL, the bytes never flow over gRPC → [`cri-api/.../api.proto`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto)
- **containerd** builds `http://127.0.0.1:<port>/exec/<token>` with a one-time token valid for 1 minute → [`containerd/.../container_exec.go`](https://github.com/containerd/containerd/blob/main/internal/cri/server/container_exec.go) → [`cri-streaming/.../server.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cri-streaming/pkg/streaming/server.go) + [`request_cache.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cri-streaming/pkg/streaming/request_cache.go)
- **Shared code:** the streaming server isn't containerd code. It lives in Kubernetes as [`k8s.io/cri-streaming`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/cri-streaming), a "staging" module that's auto-published as its own read-only repo so runtimes can import it without depending on the whole kubelet. containerd pulls it in (`k8s.io/cri-streaming v0.37.1` in its `go.mod`). It was split out of `k8s.io/kubelet/pkg/cri/streaming`, so older posts show that path.


**References:**
- [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [kubernetes/kubernetes#89899: exec over web sockets protocol is flawed](https://github.com/kubernetes/kubernetes/issues/89899)
