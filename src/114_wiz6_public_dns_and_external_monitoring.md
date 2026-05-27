# When Public DNS Hands Out Private IPs: wiz6 and Watching From Outside

## The Symptom

`wiz6.goldentooth.net` worked great. From my couch. On my own network. The moment I tried to hit it from the outside world — phone on cellular, a friend's connection, anywhere that wasn't behind my own router — it died. No connection, no timeout I could blame on TLS, just nothing.

The tell was one `dig`:

```
$ dig +short wiz6.goldentooth.net A
10.4.11.1
```

`goldentooth.net` is a *public* zone on Route53. Anyone on the internet can look up `wiz6` and they'll get an answer. The answer just happens to be `10.4.11.1`, which is RFC1918 space — non-routable, private, meaningless to anyone who isn't already inside my LAN. So the public DNS record was confidently advertising an address that only works if you don't need DNS to find it. Cool. Very useful.

## Why It Was Lying

The cluster has two completely different ways of getting records into Route53, and `wiz6` had picked the wrong one.

**external-dns** watches `Service` and `gateway-httproute` objects and publishes whatever address it sees. For a gateway-fronted service, that address is the MetalLB LoadBalancer VIP — `10.4.11.1`. external-dns has no concept of my WAN IP. It can only publish what it can see, and what it can see is private.

**The ddns-cronjob**, meanwhile, is the thing that actually works for public services. It runs every 5 minutes, asks the outside world "what's my public IP?" via `ifconfig.me`, and UPSERTs that into Route53:

```sh
PUBLIC_IP=$(curl -sf https://ifconfig.me || curl -sf https://api.ipify.org || curl -sf https://checkip.amazonaws.com)
for RECORD in pds.goldentooth.net "*.pds.goldentooth.net"; do
  ...UPSERT $RECORD -> $PUBLIC_IP...
done
```

`pds` was in that loop. `pds` was also explicitly *excluded* from external-dns (`--exclude-domains=pds.goldentooth.net`) so the two wouldn't fight over the record. `pds` worked. `wiz6` was just sitting there on external-dns, getting the private VIP stamped into the public zone over and over.

Both services attach to the *same* gateway, the same HTTPS listener, the same router port-forward. The only difference between "reachable from the internet" and "lol no" was which of these two mechanisms owned the DNS record.

## The Fix

I briefly entertained doing split-horizon DNS properly — internal resolver hands out the private VIP, public Route53 hands out the WAN IP, everyone's happy. Then I remembered there *is* no internal resolver. The netboot dnsmasq runs with `port=0` (it's DHCP/TFTP only), and the LAN's actual DNS comes from UniFi. Standing up a whole split-horizon authority to make one toy service slightly faster on the LAN is the kind of thing I'd regret in six months. So: WAN-only, exactly like `pds`. LAN clients reach it via NAT hairpin, same as they already do for everything else.

Two edits. Exclude `wiz6` from external-dns:

```diff
             - --exclude-domains=pds.goldentooth.net
+            - --exclude-domains=wiz6.goldentooth.net
```

And add it to the ddns loop:

```diff
-                  for RECORD in pds.goldentooth.net "*.pds.goldentooth.net"; do
+                  for RECORD in pds.goldentooth.net "*.pds.goldentooth.net" wiz6.goldentooth.net; do
```

Push, let Flux reconcile, wait for the cronjob to fire. external-dns stops overwriting the record (excluded domains are left untouched, not deleted — same reason `pds` persists), and ddns flips it to the WAN IP within 5 minutes.

## The Better Question: How Would I Even Know?

This whole thing was invisible to my monitoring, and that bugged me more than the outage. I have a blackbox-exporter. It probes `goldentooth.net` and `clog.goldentooth.net`. It would have reported `wiz6` perfectly **healthy** the entire time it was broken — because the prober runs *inside the cluster*, and from inside the cluster, `10.4.11.1` is completely reachable.

That's the trap: a monitor that lives inside the thing it's monitoring will happily confirm that the thing is fine, right up until it isn't, from a vantage point that doesn't matter. NAT hairpin makes it worse — an internal probe hitting the WAN IP loops back through the router on a *different path* than real external ingress, so even "probe the public IP" lies to you from the inside.

So I built two layers, each answering a different question.

### Layer One: Is the Public Record Even Sane?

A cronjob that queries *public* resolvers directly — `1.1.1.1` and `8.8.8.8`, explicitly, so it sees the internet's view and not any local override — and screams if a public name resolves to a non-routable address or fails entirely. The interesting bit is the RFC1918 classifier, in glorious POSIX `ash`:

```sh
is_public() {
  IFS=. read -r a b c d <<EOF
$1
EOF
  case "$a" in
    0|10|127) return 1 ;;
    169) [ "$b" = "254" ] && return 1 ;;
    172) [ "$b" -ge 16 ] && [ "$b" -le 31 ] && return 1 ;;
    192) [ "$b" = "168" ] && return 1 ;;
    100) [ "$b" -ge 64 ] && [ "$b" -le 127 ] && return 1 ;;
  esac
  return 0
}
```

10/8, loopback, link-local, the 172.16–31 block (and correctly *not* 172.15 or 172.32 — that off-by-one is exactly the kind of thing that bites you at 2am), 192.168, and CGNAT 100.64/10. On failure it POSTs to ntfy's `cluster-alerts` topic, where Alertmanager and Falco already shout. This catches the exact regression that just happened, and names the cause instead of just saying "site down."

### Layer Two: Can the Internet Actually Reach It?

For true external reachability you need a vantage point that is *not on your network and not in your cluster*. I went with Route53 health checks in Terraform — AWS probes the endpoint over HTTPS from its global checker fleet, a CloudWatch alarm watches each check, and the alarm notifies an SNS topic that emails me.

The design rule I care about most here: **do not route the "is the cluster reachable?" alert through the cluster.** My in-cluster cronjob posts to self-hosted ntfy, which is fine for internal faults, but if the WAN link or the gateway is down, that alert can't escape the building. SNS→email leaves AWS entirely. It can still page me when the thing being monitored is the thing that's broken.

```hcl
resource "aws_route53_health_check" "service" {
  for_each          = local.monitored_services   # { pds = "...", wiz6 = "..." }
  type              = "HTTPS"
  fqdn              = each.value
  port              = 443
  resource_path     = "/"
  request_interval  = 30
  failure_threshold = 3
}
```

One service map drives `for_each` across the health checks, the alarms, everything. Adding the next public service is a one-line edit.

## The Plot Twist: I Didn't Actually Apply Anything

I wrote the Terraform, validated it, committed it, pushed it. Then I tried to `terraform apply` like a responsible adult and got smacked:

```
Error: Error acquiring the state lock
StatusCode: 412 ... PreconditionFailed
Lock Info:
  Operation: OperationTypePlan
  Who:       runner@runnervmg397c
```

`Who: runner@...`. That's not me. That's a CI runner. Because — and I'd forgotten this — the `terraform` repo has continuous *delivery*. `continuous-delivery.yaml` runs `terraform apply` with `apply: true` on every push to `main`, plus an hourly scheduled apply for good measure. My push had *already* triggered the apply. The runner was mid-plan, holding the state lock, doing exactly what I was about to do manually and clumsily on top of it.

Breaking that lock to force my own apply would've been a great way to corrupt the state with two writers. The correct move when you see someone else holding the lock is to put the keyboard down and let them finish. So I watched instead:

```
$ gh run watch 26516226197 --exit-status
✓ Complete job
```

Pushing to this repo *is* the apply. Same mental model as Flux for the gitops repo — commit equals deploy — I'd just never internalized it for Terraform.

## Verification

Trust nothing, check the actual AWS state:

```
$ aws sns list-subscriptions-by-topic --topic-arn ...goldentooth-external-monitoring
email   alerts@goldentooth.net   PendingConfirmation

$ aws cloudwatch describe-alarms --alarm-name-prefix external-
external-pds-unreachable    OK
external-wiz6-unreachable   OK
```

Both alarms **OK** — meaning AWS's global checkers are reaching both services over HTTPS, from outside, *right now*. That's the real proof: `wiz6` isn't just resolving correctly, it's actually answering the internet. And the DNS itself, from resolvers that have nothing to do with my network:

```
$ dig +short @1.1.1.1 A wiz6.goldentooth.net
66.61.26.32
$ dig +short @8.8.8.8 A wiz6.goldentooth.net
66.61.26.32
```

`66.61.26.32`. The WAN IP. Not `10.4.11.1`. We have liftoff.

The one loose thread: that SNS subscription says `PendingConfirmation`. AWS sent an opt-in link to `alerts@goldentooth.net`, and until somebody clicks it the alarms can fire all they like into the void. Which means my shiny new external monitoring is, at this exact moment, monitoring externally and telling absolutely no one. A fitting end. I'll go click the email. Probably.
