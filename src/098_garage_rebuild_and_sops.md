# Collateral Damage: Garage, Docker Registry, and SOPS Ergonomics

## The Fallout

So the SeaweedFS migration ([chapter 97](./097_seaweedfs_migration.md)) went great. All PVCs migrated, Longhorn nuked, everyone's happy. Except for one small detail I didn't think about until things started crashing: Garage stores its entire world — cluster layout, S3 keys, bucket definitions — on those PVCs. New PVCs means new Garage. Completely blank Garage. Garage with amnesia.

The symptom was immediate: all four Garage pods went into CrashLoopBackOff. The liveness probe hits `/health` on the admin API, but a Garage node with no cluster layout returns 503 on `/health`. Pod starts, probe fires, 503, Kubernetes kills it, repeat forever. The classic "I'm not healthy because nobody's told me what health means" problem.

## Rebuilding Garage's Brain

### The Liveness Trap

First thing: stop Kubernetes from killing the pods long enough to actually configure them. Bumped the liveness probe `failureThreshold` to 100 (essentially "try for 50 minutes before giving up"):

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3903
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 100
```

Pushed, waited for pods to stabilize in Running state (still failing health checks, but not getting killed).

### Applying the Layout

Garage v2 admin API. Port-forwarded to one of the pods:

```bash
kubectl -n garage port-forward pod/garage-0 3903:3903 &
ADMIN_TOKEN=$(kubectl -n garage get secret garage-secrets -o jsonpath='{.data.admin-token}' | base64 -d)
```

First, get the node IDs. Each Garage pod generates a random node ID on first start:

```bash
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" http://localhost:3903/v2/GetClusterStatus
```

This returns each node's ID, hostname, and whether it has a role assigned (none of them did, obviously).

Now the fun part. Garage's v2 API for updating the cluster layout took me a few tries to figure out. The format is:

```bash
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3903/v2/UpdateClusterLayout \
  -d '{"roles":[
    {"id":"<node-id>","zone":"dc1","capacity":1000000000,"tags":["nvme"]},
    {"id":"<node-id>","zone":"dc1","capacity":1000000000,"tags":["nvme"]},
    {"id":"<node-id>","zone":"dc1","capacity":1000000000,"tags":["nvme"]},
    {"id":"<node-id>","zone":"dc1","capacity":1000000000,"tags":["nvme"]}
  ]}'
```

Then apply:

```bash
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3903/v2/ApplyClusterLayout \
  -d '{"version":1}'
```

After that, `/health` started returning 200, the liveness probes went green, all four pods showed Running with 0 restarts. Health check confirmed: 4 nodes connected, 256 partitions, all assigned.

Reverted the `failureThreshold` back to a sane default afterward.

## Docker Registry: The Other Casualty

With Garage rebuilt from scratch, the docker-registry deployment was the next domino. It stores container images in a Garage S3 bucket called `docker-registry`. That bucket? Gone. The S3 access key? Also gone. CrashLoopBackOff.

The fix was mechanical — recreate everything through the Garage v2 admin API:

```bash
# Create a new S3 key
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3903/v2/CreateKey \
  -d '{"name":"docker-registry"}'

# Create the bucket
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3903/v2/CreateBucket \
  -d '{"globalAlias":"docker-registry"}'

# Grant permissions
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:3903/v2/AllowBucketKey \
  -d '{"bucketId":"<bucket-id>","accessKeyId":"<key-id>","permissions":{"read":true,"write":true,"owner":true}}'
```

Patched the Kubernetes secret with the new credentials, restarted the deployment. Registry came up 1/1, 0 restarts.

But now the SOPS-encrypted secret in Git still had the old credentials. Next Flux reconciliation would overwrite the working K8s secret with dead creds. Needed to decrypt, update, re-encrypt.

## The SOPS Problem

```
$ sops decrypt infrastructure/docker-registry/registry-s3-secret.yaml
Failed to get the data key required to decrypt the SOPS file.
```

Right. The age private key. It's in 1Password. I know this because `cluster/talenv.yaml` already has:

```yaml
SOPS_AGE_KEY_CMD: 'op read op://Goldentooth/talhelper_age_key/password'
```

That's talhelper's native support for fetching the key from 1Password. But `SOPS_AGE_KEY_CMD` is a talhelper thing, not a SOPS thing. SOPS itself wants `SOPS_AGE_KEY` or `SOPS_AGE_KEY_FILE` as environment variables. And there are actually *two* age keys — one for `cluster/` (talhelper) and one for `gitops/` (Flux). Two different recipients, two different private keys, both in 1Password.

Previously I was just copy-pasting the key from 1Password into the terminal every time I needed to decrypt something. This is, technically, a workflow. It is not a *good* workflow.

## The Fix: direnv + 1Password CLI

Installed direnv, created `.envrc` at the project root:

```bash
# Pull SOPS age keys from 1Password automatically.
# Both keys are needed: talhelper for cluster/, flux for gitops/.
TALHELPER_KEY="$(op read op://Goldentooth/talhelper_age_key/password)"
FLUX_KEY="$(op read op://Goldentooth/flux_age_key/password)"
export SOPS_AGE_KEY="$TALHELPER_KEY
$FLUX_KEY"
```

`SOPS_AGE_KEY` supports multiple keys separated by newlines. SOPS tries each one until it finds a match. So now `cd`-ing into the project directory automatically loads both keys from 1Password, and `sops decrypt` just works against any encrypted file in either subdirectory.

```
$ cd ~/Projects/goldentooth
direnv: loading ~/Projects/goldentooth/.envrc
direnv: export +SOPS_AGE_KEY

$ sops decrypt gitops/infrastructure/docker-registry/registry-s3-secret.yaml
apiVersion: v1
kind: Secret
...
    accesskey: GK42ea6fa0e627dcb58e7cef67
    secretkey: <redacted>

$ sops decrypt cluster/talsecret.sops.yaml
cluster:
    id: <redacted>
...
```

No more copy-pasting keys from 1Password. The `op read` command handles authentication through the 1Password desktop app or Touch ID, depending on your setup. The `.envrc` is in `.gitignore` so it doesn't get committed.

Updated the SOPS secret with the new Garage credentials, committed, and now Flux won't blow away the working credentials on its next reconciliation.
