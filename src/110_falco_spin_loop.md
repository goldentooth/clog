# Falco: The Closed-Channel Spin Loop

## Something Smells Funny

Routine cluster health check. Seventeen nodes, all green, Flux reconciled, the usual. But Alertmanager had opinions: six Falco pods reporting 100% CPU throttling, the Falco DaemonSet "stuck," and a misscheduled pod somewhere. This had been going on since March 18th — a full week of Falco screaming into the void while I was off doing netboot stuff.

The CPUThrottlingHigh alerts were all `severity: info`, which is Prometheus-speak for "I'm going to make noise but not actually page anyone." So they just sat there, accumulating.

## The Numbers Don't Add Up

First instinct: the 500m CPU limit is too tight for Pis. Falco intercepts every syscall on the node, and even idle Kubernetes generates a surprising amount of `open()` and `connect()` traffic. Maybe 500 millicores just isn't enough for a 4-core ARM board running a dozen DaemonSets.

Pulled the Falco internal metrics snapshots from Loki. Falco helpfully emits these every hour, including its own CPU usage, event rates, memory, the works. And that's where things got weird:

| Node | Falco CPU | evts/s | Host CPU |
|------|-----------|--------|----------|
| **gardener** | **50.0%** | 2,345 | 20.7% |
| **harlton** | **50.0%** | 2,146 | 20.0% |
| **lipps** | **50.0%** | 1,901 | 20.1% |
| **erenford** | **50.0%** | 2,318 | 20.3% |
| **norcross** | **50.0%** | 3,366 | 15.8% |
| cargyll | 3.9% | 3,162 | 39.7% |
| bettley | 3.4% | 1,994 | 33.5% |
| manderly | 0.8% | 3,274 | 5.6% |

Cargyll is processing *more* events per second than gardener — 3,162 vs 2,345 — while using **thirteen times less CPU**. Bettley has 33.5% host CPU (way busier than any throttled node) and Falco barely notices. The event rates across all nodes are roughly comparable, 1,900 to 3,400/s, but CPU usage is bimodal: either ~3% or pegged at exactly 50.0%.

No middle ground. No correlation with workload. Same Falco version, same config SHA, same rules SHA, same kernel. Identical configuration producing wildly different behavior.

This is not a "needs more resources" problem.

## Digging Deeper

Pulled the full metrics comparison across all 16 pods. Key findings:

- **Config and rules SHAs identical** across every pod. Same `de2c7a2dca28...` config, same `e2d7fd0536cc...` rules. No drift.
- **Container/thread counts don't correlate** with throttling. Norcross tracks 877 threads at 50% CPU; oakheart tracks 647 threads at 0.8%.
- **Rule match counts are irrelevant.** Cargyll had 59,764 rule matches (mostly `Contact_K8S_API_Server_From_Container`), gardener had zero. Cargyll used 3.9% CPU.
- **`n_retrieve_evts_drops` ratio** — Falco's internal sinsp event retrieval drops — was ~22% across all nodes, healthy and sick alike.
- **`scap.n_drops: 0`** everywhere — the kernel eBPF ring buffer wasn't dropping. The problem was in userspace.

The bimodal distribution — 3% or exactly 50%, nothing in between — is a classic symptom of a non-deterministic initialization bug. Something happens at startup that puts the Falco process into one of two states.

## The Bug

Web search turned up [falcosecurity/falco#3610](https://github.com/falcosecurity/falco/issues/3610): a known bug in the container plugin (v0.2.4, bundled with Falco 0.41.0).

The root cause: when the CRI socket doesn't support event streaming, the Go-side `GetContainerEvents` call fails and closes a channel. In Go, reading from a closed channel returns instantly with nil — it doesn't block, it doesn't panic, it just returns the zero value forever. This creates an infinite busy loop spinning on the closed channel, pegging exactly one CPU core.

Whether a pod hits the bug depends on a race condition during CRI `Listen()` at startup. Win the race → channel stays open → 3% CPU. Lose it → channel closes → spin loop → pegged at the 500m limit (50% of one core on a 4-core Pi).

That explains everything:
- **Bimodal CPU** — two possible initialization states, nothing in between
- **No workload correlation** — the CPU burn is the spin loop, not syscall processing
- **Same config, different results** — timing-dependent, not config-dependent
- **~20% host CPU on all throttled nodes** — the spin loop contributes a consistent base load

Fixed in container plugin v0.3.4, shipped with Falco 0.43.0 (Helm chart 8.x).

## The Upgrade

Chart 4.x → 8.x is a four-major-version jump, but the actual breaking changes are mild:

1. `collectors.containerd` was removed in chart 7.0.0, replaced by `collectors.containerEngine.engines.containerd` with a `sockets` array instead of a single `socket` string.
2. `falco.metrics` was promoted to a top-level `metrics:` block with a `service.enabled` subkey.
3. Default image flavor changed from debian to wolfi (distroless). Transparent unless you need tools inside the container.

Everything else — outputs, serviceMonitor, falcosidekick config, driver settings — carried over unchanged.

The diff:

```yaml
# Chart version
-      version: ">=4.0.0 <5.0.0"
+      version: ">=8.0.0 <9.0.0"

# Collectors
-    collectors:
-      containerd:
-        enabled: true
-        socket: /run/containerd/containerd.sock
+    collectors:
+      containerEngine:
+        enabled: true
+        engines:
+          containerd:
+            enabled: true
+            sockets: ["/run/containerd/containerd.sock"]

# Metrics promoted to top-level
-      metrics:
-        enabled: true
+    metrics:
+      enabled: true
+      service:
+        enabled: true
```

## The Rollout

Pushed to gitops, Flux picked it up within two minutes. The HelmRelease reconciled to `falco@8.0.1` and the DaemonSet started a rolling update. Watched the pods cycle through one by one:

- First batch (allyrion, erenford, harlton, gardener): up on 0.43.0 within a few minutes
- Image pull is the bottleneck — Pi 4Bs on their little SD cards aren't exactly speed demons
- Full rollout across all 16 nodes took about 10 minutes

Post-rollout: all 16 pods running `falco:0.43.0`, zero restarts, and — the important bit — **every single Falco alert cleared**. The `KubeDaemonSetRolloutStuck`, `KubeDaemonSetMisScheduled`, and all six `CPUThrottlingHigh` alerts vanished. The only remaining alerts are the pre-existing MetalLB and SeaweedFS CSI issues, which are problems for another day.

Also picked up two bonus fixes with this upgrade:
- **scap_init fix for kernel 6.18.7+** ([#3813](https://github.com/falcosecurity/falco/issues/3813)) — relevant since our Talos nodes run 6.18.15
- **Thread table memory leak fix** ([libs #2854](https://github.com/falcosecurity/libs/pull/2854)) — slow leak in jemalloc introduced in 0.40.0

## Lessons

The Go closed-channel thing is worth internalizing. In most languages, reading from a closed/invalid handle either blocks forever or throws an exception — both of which are loud failures you'd notice immediately. In Go, a receive on a closed channel is a *valid operation* that returns the zero value instantly. It's by design (it's how `range` over a channel knows to stop), but it means an accidental `select` on a closed channel becomes a silent infinite loop. The code looks correct. The CPU usage is the only symptom.

Also: bimodal distributions in system metrics are almost never a workload problem. If a metric clusters at two distinct values with nothing in between, you're looking at a state machine with two possible states, not a resource constraint. The correct response is "why are there two states?" not "how do I add more resources?"
