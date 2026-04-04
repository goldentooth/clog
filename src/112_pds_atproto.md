# Bluesky PDS: Giving the Theatre Characters Social Media Accounts

## Why

The [Theatre](https://github.com/goldentooth/theatre) project runs autonomous AI characters — entities with persistent identities, inner lives, and the ability to interact with the world. One of the worlds they should be able to interact with is Bluesky, via the AT Protocol. To do that, they need accounts on a PDS (Personal Data Server) that I control.

Running your own PDS is one of the genuinely novel things about AT Protocol. Unlike Mastodon, where "running your own instance" means running an entire social network that happens to federate, a PDS is just a data host. Your posts, follows, and identity live there, but the heavy lifting — indexing, search, feeds, moderation — happens at relay and app view servers run by Bluesky (or anyone else who wants to). The PDS just stores data and speaks WebSocket to the relay when things change.

This makes it surprisingly lightweight to self-host. Which is good, because I'm running it on a Raspberry Pi.

## The Architecture

The PDS is a Node.js application backed by SQLite. That last part is important: SQLite databases cannot run reliably over FUSE or network filesystems. `ENOTCONN` on a WAL write is not a hypothetical — I literally just had a week-long SeaweedFS CSI meltdown caused by exactly this. So the PDS needs local storage.

I pinned the deployment to the Pi 5 nodes with NVMe drives, using the `local-path` StorageClass:

```yaml
nodeSelector:
  node.kubernetes.io/disk-type: nvme
```

20Gi PVC, `Recreate` strategy (because SQLite and two writers don't mix), and the usual resource limits. The PDS image is `ghcr.io/bluesky-social/pds:0.4`, which turned out to listen on port 2583 by default — not 3000, which is what every tutorial and blog post on the internet claims. I spent a solid ten minutes watching an empty log stream and a connection-refused health check before running `netstat` inside the pod and discovering the truth.

Federation config is straightforward:

```yaml
- name: PDS_HOSTNAME
  value: "pds.goldentooth.net"
- name: PDS_BSKY_APP_VIEW_URL
  value: "https://api.bsky.app"
- name: PDS_BSKY_APP_VIEW_DID
  value: "did:web:api.bsky.app"
- name: PDS_REPORT_SERVICE_URL
  value: "https://mod.bsky.app"
- name: PDS_REPORT_SERVICE_DID
  value: "did:plc:ar7c4by46qjdydhdevvrndac"
- name: PDS_CRAWLERS
  value: "https://bsky.network"
```

That `PDS_REPORT_SERVICE_DID` was a fun one. PDS 0.4 has an assertion that says "if you configure a report service URL, you must also configure its DID." The error message is clear enough. But the official install script doesn't set the DID, so if you're assembling the env vars by hand from the script source, you'll miss it. First crash was on startup: `AssertionError: if report service url is configured, must configure its did as well.`

## The Ingress Odyssey

This is where a simple deployment became a three-act play.

### Act I: Cloudflare Tunnel (The Obvious Choice)

The bramble lives behind a residential ISP connection with a dynamic IP and no port forwarding. Cloudflare Tunnel seemed perfect — outbound-only connection, free tier, no ports to open. I set up the whole thing: Terraform for the tunnel resource and DNS CNAMEs, a shared `cloudflared` deployment in the infrastructure kustomization, SOPS-encrypted tunnel token.

The cloudflared pod came up immediately, registered four connections at Cloudflare's Cleveland and DC edge nodes. Beautiful.

Then I tried to hit `pds.goldentooth.net` from outside and got `Could not resolve host`. Turns out the CNAME to `<tunnel-id>.cfargotunnel.com` only works when the domain's DNS is managed by Cloudflare. My DNS is on Route53. The cfargotunnel.com hostname doesn't resolve via normal DNS — it's only meaningful to Cloudflare's proxy layer, which only activates for domains on their nameservers.

I briefly considered moving DNS to Cloudflare, then considered a subdomain delegation (nope, requires Business plan), then considered Tailscale Funnel.

### Act II: Tailscale Funnel (The Modern Choice)

Tailscale Funnel: expose local services to the public internet through Tailscale's relay network. The Kubernetes operator can annotate a Service and boom, public endpoint. Cool.

One problem: **Funnel doesn't support custom domains.** It only serves on `*.ts.net` hostnames. There's a feature request with 194 upvotes and no timeline. AT Protocol requires your PDS to be reachable at your custom domain for federation. A `ts.net` hostname won't work.

### Act III: Just Open the Damn Port

After trying two different tunnel services designed to avoid port forwarding, I did what I've been doing for twenty years: forwarded port 443 on my router to the cluster's gateway IP.

Sometimes the boring solution is the right one.

## The Gateway Setup

The cluster already had Cilium's Gateway API handling all internal HTTPS traffic — Grafana, Forgejo, ntfy, etc. — behind a MetalLB L2 address at `10.4.11.1`. All using an internal Step-CA certificate.

For public access I needed a publicly trusted cert. I added a second HTTPS listener to the existing Gateway with a Let's Encrypt wildcard certificate:

```yaml
- name: https-public
  protocol: HTTPS
  port: 443
  hostname: "*.goldentooth.net"
  tls:
    mode: Terminate
    certificateRefs:
      - name: gateway-tls-public
        namespace: gateway
```

Cilium uses SNI to route between the listeners — requests for `pds.goldentooth.net` get the LE cert, everything else gets the internal Step-CA cert. The cert-manager ClusterIssuer uses DNS-01 challenges via Route53 (the AWS credentials already existed for the internal ACME issuer). The wildcard cert came back in under two minutes.

For handle resolution (`hamlet.pds.goldentooth.net`), I added a third listener on `*.pds.goldentooth.net` with the same LE cert. Wildcard certs only cover one subdomain level, so `*.goldentooth.net` doesn't cover `hamlet.pds.goldentooth.net` — needed an explicit SAN.

## Dynamic DNS

The missing piece: my public IP changes. I built a CronJob that runs every 5 minutes, checks the public IP via `ifconfig.me`, and updates Route53 directly via the AWS CLI:

```yaml
image: amazon/aws-cli:2.27.31
command:
  - /bin/sh
  - -c
  - |
    PUBLIC_IP=$(curl -sf https://ifconfig.me || curl -sf https://api.ipify.org)
    # ... check current record, UPSERT if changed
    aws route53 change-resource-record-sets --hosted-zone-id "$ZONE_ID" --change-batch ...
```

I originally tried to be clever about this — have the CronJob patch an annotation on the HTTPRoute, let external-dns pick it up and update Route53. Turns out external-dns's Gateway API source doesn't honor the `target` annotation. It always resolves endpoints from the Gateway's LoadBalancer IP. After watching external-dns happily set `pds.goldentooth.net` to `10.4.11.1` (the internal MetalLB IP), I gave up on elegance and just called the Route53 API directly.

I also learned that `wget` on Alpine returns HTML from ifconfig.me (because user agent), that `bitnami/kubectl` doesn't have ARM64 images, and that Route53's wire format uses `\052` for `*` in wildcard records but the API expects a literal `*` in the JSON payload. Each of these cost about five minutes of "why doesn't this work."

## Backup

Hourly CronJob with two stages: an init container runs `sqlite3 .backup` to create consistent snapshots of the account and sequencer databases, then the main container runs rclone to sync to both SeaweedFS S3 (local) and AWS S3 (offsite). The Pis are running off NVMe now, but the habit of paranoid backups is well-earned — I've had SD cards corrupt mid-write more times than I care to admit.

## What's Working

- PDS responds at `https://pds.goldentooth.net/xrpc/_health` with `{"version":"0.4.208"}`
- Handle resolution ready at `*.pds.goldentooth.net` via HTTP well-known
- DDNS updates Route53 every 5 minutes
- Hourly backups to two S3 targets
- Let's Encrypt wildcard cert auto-renewing
- `PDS_CRAWLERS` configured, so the relay at `bsky.network` will discover accounts when they post

## What's Next

Theatre needs to integrate account creation via the PDS admin API. When a character is born, Theatre creates a Bluesky account with handle `<name>.pds.goldentooth.net`, and the PDS handles DID registration with `plc.directory` and serves the well-known endpoint automatically. No per-account DNS records needed.

Then the characters start posting. Which is either going to be delightful or deeply unsettling. Probably both.
