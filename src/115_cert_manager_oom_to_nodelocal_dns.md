# It's Always DNS, Eventually: From an Expired Cert to NodeLocal DNS

This one started as a one-line "hey, the MCP image scan is failing" and turned into a five-yak shave that ended three subsystems away from where it began. Cert-manager, the entire internal PKI, an ACME challenge that had been dead since March, and finally — because the universe has a sense of humor — DNS. It's always DNS. It's just that this time DNS was wearing a trenchcoat labeled "expired TLS certificate."

## Yak Zero: The Registry Scan

Flux's image automation couldn't scan the MCP registry:

```
scan failed: Get "https://registry.goldentooth.net/v2/": tls: failed to verify
certificate: x509: certificate has expired or is not yet valid: current time
2026-06-02T20:04:50Z is after 2026-05-26T07:05:44Z
```

Expired cert. Fine, cert-manager will reissue it, right? Except the `registry-tls` Certificate said `Ready=True` while being very much expired. So did `gateway-tls`. So did the pki test certs. *Every* `step-ca`-issued cert had expired at almost the same moment, `2026-05-25`-ish, and none of them had renewed.

The one cert that was fine? `gateway-tls-public` — the 90-day Let's Encrypt wildcard. It wasn't due to renew until June 3rd, so it hadn't *noticed* anything was wrong. Which is exactly why nobody had noticed anything was wrong: every public-facing service kept serving TLS off the LE wildcard while the entire internal short-lived PKI quietly turned into a pumpkin.

When a pile of 24h certs all freeze at the same timestamp, you don't have a certificate problem. You have an *issuer* problem. Or, as it turned out, an issuer's-best-friend problem.

## Yak One: The Hungry Controller

step-ca itself was up and healthy. step-issuer, fine. The PKI Flux Kustomizations, ready. The thing that was *not* fine:

```
cert-manager-cert-manager-6878f64565-zzwg9   2234   OOMKilled   137
```

Two thousand two hundred and thirty-four restarts. Exit 137 — SIGKILL, OOM. The cert-manager controller was in a tight crash loop, dying ~4 seconds after start, and a controller that can't stay alive can't renew anything. The short-lived step-ca certs expired within a day; the LE wildcard hid it.

The limit was `128Mi`. Steady-state usage, once I got it up long enough to look, was ~67Mi. So it wasn't a steady leak — it was a *startup spike*. The logs gave it away:

```
Trace[...]: ---"Objects listed" 15162ms
```

On boot, the controller lists every `CertificateRequest` into its informer cache. There were **4,619** of them. That list-all spike blew past 128Mi before cache sync finished, the kernel killed it, and it never reached the reconcile loop — a perfect self-reinforcing trap, because the thing that would have garbage-collected the backlog was the controller that couldn't boot.

Why 4,619 CRs? `revisionHistoryLimit` was unset on every Certificate, which for cert-manager means *keep all revisions forever*. The `canary-certificate` renews **every hour** (it's a deliberate "is the PKI alive" canary), so over ~200 days it had quietly minted ~3,600 CertificateRequests that nobody ever cleaned up.

Two fixes, both in gitops:

```diff
# cert-manager HelmRelease
     limits:
-      memory: 128Mi
+      memory: 512Mi
```

```diff
# every Certificate
   secretName: registry-tls
+  revisionHistoryLimit: 3
```

Bumped the limit first (kubectl for immediate relief, then committed), watched the controller come up and *stay* up, then applied `revisionHistoryLimit: 3` and watched the revision-manager drain the backlog: 4,619 → 18. Amusingly, during the bulk GC the controller briefly hit **139Mi** — above the old 128Mi limit. So the cleanup itself would have re-OOMed it at the old setting. The memory bump *had* to come first. Order of operations, baby.

Certs reissued automatically, the registry scan went green, and that should have been the end of it.

It was not the end of it.

## Yak Two: The Cert That Died in March

While poking the PKI I noticed `acme-test-certificate` had been `Ready=False` since `2026-03-17`. Its sibling `dns01-test-certificate` — same `step-ca-acme` issuer, but DNS-01 instead of HTTP-01 — was perfectly healthy. So the ACME *machinery* worked; specifically the HTTP-01 path didn't, and hadn't for months.

This is where I should have been suspicious that "broken since March" and "cert-manager OOM since late May" were two different bugs. I was not yet suspicious. I went down the hole.

The challenge errored with:

```
HTTP-01: unexpected non-ACME API error: context deadline exceeded
```

and step-ca's own logs, eventually:

```
"type":"http-01","error":{"type":"urn:ietf:params:acme:error:connection",
"detail":"The server could not connect to validation target"}
```

I burned a genuinely embarrassing amount of time here. I confirmed the gateway's `:80` listener does a blanket HTTP→HTTPS 301 redirect (bad for ACME, true, but not the cause). I watched the solver pod take ~30s to cold-start its image on the Pi, during which the gateway correctly 503s because there's no backend yet. I convinced myself there was a Cilium hairpin asymmetry because a LAN `curl` got `200` while step-ca got `000` — then proved that wrong by hitting the same URL from both at the *same instant* and getting identical results. The "asymmetry" was just me testing at different points in the solver's lifecycle. Classic.

The actual tell, when I finally read it instead of theorizing: `curl` from step-ca's pod returned exit code **6**. Couldn't resolve host. Not a connection timeout — *DNS*. step-ca couldn't reliably resolve `acme-test.goldentooth.net` to the gateway, intermittently, and HTTP-01 is the only ACME flow that needs an in-cluster component to resolve a `*.goldentooth.net` name. DNS-01 just writes a TXT record and asks Route53.

I retired the HTTP-01 test (DNS-01 already proves the ACME path, and nothing real uses HTTP-01), left a comment in the kustomization explaining *why* so future-me doesn't "fix" it back in, and turned to the actual monster.

## Yak Three: It Was DNS

Here's the thing that took embarrassingly long to accept: resolution of `*.goldentooth.net` from inside the cluster was *intermittent*. Sometimes 10/10, sometimes 2/10, for the same name, same upstream. And `example.com` was rock solid the whole time.

I almost convinced myself it was name-specific — `goldentooth.net` is a public Route53 zone whose records point at *private* IPs (the gateway `10.4.11.1`, the registry `10.4.11.8`, the control-plane VIP `10.4.0.9`; see entry 114 for the full "public DNS handing out RFC1918" saga). Maybe a rebinding filter? No. The flaky readings all happened while the cluster was thrashing — OOM recovery, 4,600-CR garbage collection, solver pods churning. Under load. The calm readings were all 10/10.

CoreDNS told me the rest itself:

```
[ERROR] plugin/errors: ... read udp 10.244.3.175:xxxxx->8.8.8.8:53: i/o timeout
```

…repeated for github.com, route53.amazonaws.com, registry.goldentooth.net — *everything* external, not just goldentooth. And the metrics:

```
coredns_forward_healthcheck_broken_total   5181
coredns_dns_responses_total{rcode="SERVFAIL"}   5199
coredns_dns_responses_total{rcode="NXDOMAIN"}    13700000
coredns_dns_responses_total{rcode="NOERROR"}      5400000
coredns_dns_requests_total{type="A"}     9582595
coredns_dns_requests_total{type="AAAA"}  9581004
coredns_cache_hits_total ~1.26M   coredns_cache_misses_total ~17.9M
```

Read that NXDOMAIN number again. **13.7 million** NXDOMAINs versus 5.4M NOERROR — a 2.5:1 ratio of *wasted* lookups. That's `ndots:5` doing its thing: every external name gets tried as `name.svc.cluster.local`, `name.cluster.local`, etc. first (×A and ×AAAA), all NXDOMAIN, before the real query. ~93% cache miss rate. Two CoreDNS replicas. No node-local cache. So the cluster generates a firehose of UDP queries to public resolvers, and under load that firehose overruns CoreDNS's own egress (conntrack/SNAT pressure on its source ports), the UDP replies drop, and resolution of *any* externally-forwarded name — including the internal-but-public `*.goldentooth.net` — gets flaky.

Direct pod→`8.8.8.8` was fine. It's CoreDNS's *high-volume* egress that buckles. The records were never the problem; the path to fetch them was.

## The Fixes

**One: stop sending internal names to the internet.** `goldentooth.net` resolves correctly at the LAN resolver (the UniFi box at `192.168.1.1` — the same "there is no internal resolver, UniFi does it" reality from entry 114). So forward just that zone there, ahead of the public catch-all:

```
forward goldentooth.net 192.168.1.1
forward . 8.8.8.8 1.1.1.1 { max_concurrent 1000 }
```

A blanket `*.goldentooth.net → 10.4.11.1` rewrite would have been a disaster — those names map to *different* IPs, and `cp.k8s.goldentooth.net` is the literal Kubernetes API endpoint. Forwarding to the LAN resolver gets the correct answer for all of them, local hop, no internet round-trip.

This is Talos-managed CoreDNS, so I did it "declaratively" via a talconfig `inlineManifest` — and learned that **Talos won't overwrite a coredns ConfigMap that already exists**. `talosctl get manifests` showed it registered as `Manifest/99-coredns-corefile`, but the live ConfigMap's `managedFields` never gained a new `talos` write. The manual `8.8.8.8` edit from March had survived precisely because Talos applies CoreDNS *once* at bootstrap and then leaves it alone. So the inlineManifest is the bootstrap/DR seed; the live cluster also needed a direct `kubectl apply`. Fine. Documented. Moving on.

**Two: NodeLocal DNSCache.** A per-node CoreDNS cache that pods hit instead of crossing the network to one of two central pods. On a `kubeProxyReplacement` cluster you can't use the usual iptables/link-local trick, so it's a `CiliumLocalRedirectPolicy` (Cilium already had `enable-local-redirect-policy=true`) that transparently redirects kube-dns traffic to the local cache pod. The cache forwards all misses via `force_tcp` to a *separate* `kube-dns-upstream` Service (static ClusterIP `10.96.0.11`, same backends as kube-dns) so it reaches the real CoreDNS without the LRP catching it in a loop. CoreDNS stays the single owner of the `goldentooth.net→LAN` policy; the cache is dumb on purpose.

Two self-inflicted wounds during rollout, both caught safely:

- The DaemonSet came up `0/17` ready. The pods were *fine* — CoreDNS 1.12.3 loaded config, zero restarts — but my readiness probe pointed at `:8080`. That's the `health` plugin. The `ready` plugin lives on **`:8181`**. 404 forever. Because Cilium LRP only redirects when a node has a *ready* local backend, this was completely harmless: every node just fell back to central kube-dns the whole time I was being an idiot about port numbers.
- I tried `autopath @kubernetes` to kill the NXDOMAIN amplification at the source. It loaded without error and did *nothing* — the NXDOMAIN ratio didn't budge. `autopath @kubernetes` needs `pods verified`, and this Corefile runs `pods insecure`. So I ripped it out; the per-node cache absorbs the amplification anyway by caching the denials locally.

The CoreDNS image is distroless, by the way, so you can't `kubectl exec ... sh` into the cache pods to poke them. Learned that the fun way too.

**Three: scale the central CoreDNS.** Now that the node caches absorb the bulk of traffic, the central pods are barely working (10m CPU, 71Mi each). Bumped them 2→3 replicas for redundancy across the 17 nodes and gave them real memory headroom (`limits: 256Mi`, `requests: 128Mi` — the old `70Mi` request was below actual usage, which is its own kind of lie).

## Where It Landed

```
mcp ImageRepository:  READY=True  successful scan: found 2 tags
cert-manager:         0 restarts, 49Mi / 512Mi
CertificateRequests:  18  (was 4,619)
node-local-dns:       17/17 ready, 0 restarts
DNS reliability:      20/20 for mcp / registry / github / cluster names
broken_healthchecks:  0  (was ~5,150 per pod)
```

The `20/20` is the one that matters — that's the exact test that returned `2/10` at the start.

So: a registry scan failure was a cert-manager OOM was an unbounded-CertificateRequest-backlog, and separately a months-dead ACME cert was intermittent DNS was CoreDNS-upstream-overload was `ndots:5`-amplification-plus-no-node-cache. Every yak had a yak under it. The one consolation is that it really was, in the end, always DNS — I'd hate to break the streak.
