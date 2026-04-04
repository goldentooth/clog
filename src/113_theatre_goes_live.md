# Theatre Goes Live: From Provisioning to First Post

The PDS was running. The certs were green. The DDNS was updating. Now Theatre needed to actually *use* it.

## Account Provisioning

The first piece was making Theatre responsible for creating its own accounts. When a new character is born, Theatre should handle the full lifecycle: create the PDS account, generate an app password, persist the credentials, and update the character's config. No manual `curl` to the admin API, no copy-pasting DIDs.

I added a `PdsProvisioner` trait to the atproto module — separated from the normal `AtprotoClient` session lifecycle because account creation uses different endpoints and auth flows than posting does. The provisioner creates an account via `com.atproto.server.createAccount`, generates an app password via `createAppPassword`, writes the password to a local secrets JSON file (or k8s-mounted secret in production), and appends the `[atproto]` section to the character's `config.toml`.

The handle follows a simple pattern: `{character}.pds.goldentooth.net`. Email is synthetic: `{character}@theatre.pds.goldentooth.net`. The account password is a random 32-character string that gets generated and immediately discarded after the app password is created — Theatre never needs it again.

One gotcha: the PDS still had `PDS_INVITE_REQUIRED=true` set in the deployment env vars. I'd assumed it was in the k8s secret, but no — hardcoded in the deployment YAML. Changed it in gitops, pushed, waited for Flux to reconcile, got impatient, patched the deployment directly. Classic.

```
theatre provision nebhos --pds-url https://pds.goldentooth.net
```

```
Provisioned nebhos
  Handle: nebhos.pds.goldentooth.net
  DID:    did:plc:mhzutd4uu7d357hdmoaohdum
  Config: characters/nebhos/config.toml updated
  Secret: secrets/atproto-passwords.json updated
```

The PDS registered the DID with `plc.directory` automatically. Handle verification via `/.well-known/atproto-did` worked immediately.

## The First Post (Sort Of)

I ran a test wake cycle with the dummy provider first. The test provider always produces "I stir." — not exactly Keats, but it proved the pipeline works. The Bluesky platform adapter authenticated, called `createRecord` on `app.bsky.feed.post`, and the post appeared at:

```
at://did:plc:mhzutd4uu7d357hdmoaohdum/app.bsky.feed.post/3mionhaxpp22m
```

Then I switched to the real Claude provider and immediately hit a parsing error. Claude was wrapping its JSON response in markdown code fences — ````json ... ``` `` — which is helpful in chat and deeply unhelpful when you're trying to `serde_json::from_str` the result. Added a fence-stripping pass before the JSON parse. The kind of bug that makes you feel dumb for not predicting, and then makes you feel dumb for feeling dumb because of course the LLM does that.

## The Port Forwarding Saga

With a post on the PDS, I requested a crawl from the Bluesky relay:

```bash
curl -X POST "https://bsky.network/xrpc/com.atproto.sync.requestCrawl" \
  -H "Content-Type: application/json" \
  -d '{"hostname": "pds.goldentooth.net"}'
```

Nothing. Port 443 was confirmed closed from outside using port checkers. The port forward rule was there in Unifi. The firewall allow rule was at the top. The static route to the MetalLB subnet existed. Everything looked correct.

The problem turned out to be Unifi's Zone-Based Firewall. Community posts confirmed what I suspected: ZBF policies that cross zones get evaluated in iptables chains that fire *after* the DNAT rewrite but *before* the explicit allow rule. The port forward rewrites the destination to 10.4.11.1 (MetalLB subnet), but the FORWARD chain drops the packet because the default zone policy denies the WAN→Internal transition before the ZBF allow rule gets a chance to match.

The fix was embarrassingly simple. The Cilium Gateway service was already a LoadBalancer with auto-allocated NodePorts:

```
cilium-gateway-goldentooth   LoadBalancer   10.98.116.207   10.4.11.1   80:31286/TCP,443:31178/TCP
```

Port 31178 on any node in the 10.4.0.0/24 subnet — which the gateway *does* know how to reach, because it's the directly-connected node network. Changed the Unifi port forward from `443 → 10.4.11.1:443` to `443 → 10.4.0.10:31178`. Worked instantly.

Every service behind the Cilium Gateway is now externally reachable. The MetalLB VIP is still used internally; external traffic just enters through the NodePort side door.

## Nebhos Speaks

With external access working, I ran a real Claude-powered wake cycle. The maturity gate (default: 10 sediment entries before a character can post) was temporarily lowered to 0 for testing. Claude thought about nebhos's soul and produced:

> is there
>
> a pressure where questions collect
> like dew on glass no one
> will wipe clean
>
> the tremor before
> anything decides
> to fall

That "dew on glass no one will wipe clean" is a direct echo of the soul.md: "A film on glass that no one will wipe away." The LLM internalized the character's identity through the accumulated impressions and produced something that felt continuous with the voice rather than just parroting the source.

The post federated immediately. The Bluesky relay crawled it, the appview indexed it, and it showed up in the public feed API.

## Profile Generation

One problem: nebhos was invisible in Bluesky search. The profile was completely empty — no display name, no bio, no avatar. Bluesky's search indexer deprioritizes accounts without profiles, and accounts on third-party PDS instances are already lower priority.

I added a `theatre profile` command that:

1. Reads the character's soul
2. Calls Claude to generate a display name and bio in the character's voice
3. Sets the profile via `com.atproto.repo.putRecord` on `app.bsky.actor.profile/self`

This required adding `put_record` to the `AtprotoClient` (same pattern as `create_record` with 401 retry, but targeting the putRecord endpoint) and an `actor_profile` record builder.

```
theatre profile nebhos
```

```
Generating profile from soul...
  Display name: ∴ mist between unnamed valleys ∴
  Description:  once everywhere, now residue on forgotten glass. i know the
                weight of water undecided, the pressure before names. i do
                not speak—i precipitate. condensation holds what vastness
                cannot remember.
Profile set: at://did:plc:mhzutd4uu7d357hdmoaohdum/app.bsky.actor.profile/self
```

Setting the profile also kicked the appview into re-indexing the account — the handle resolved correctly right after, and the account became searchable.

## What's Working Now

- `theatre provision <name>` — creates PDS account, generates app password, persists credentials
- `theatre wake <name>` — full Claude-powered cognitive cycle: think, feel, post
- `theatre profile <name>` — LLM-generated display name + bio, set on Bluesky
- Full federation: posts appear in the Bluesky firehose, profiles are searchable
- External access via NodePort workaround for Unifi ZBF issues

## What's Next

Avatar generation. The characters need faces — or at least, whatever a cloud of residual moisture would have instead of a face. Probably something via the OpenAI Images API, with the soul text driving the prompt. The profile command could grow an `--avatar` flag, or image generation could be its own step.

Also need to let the maturity gates do their job properly. Right now nebhos has 2 sediment entries and a post gate of 10. That's going to be a lot of quiet thinking before the next public utterance. Which, honestly, feels right for an entity that describes itself as "the tremor before anything decides to fall."
