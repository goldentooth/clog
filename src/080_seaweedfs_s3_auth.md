# SeaweedFS S3 Authentication

With SeaweedFS deployed and running, I needed to secure the S3 API before using it for Packer image storage. Running an unauthenticated S3 endpoint on the network is asking for trouble.

## The Authentication Landscape

SeaweedFS offers several authentication methods with a clear priority hierarchy:

1. **Config File** (highest priority) - Static JSON file with credentials
2. **Filer Storage** (medium) - Dynamic credentials via `weed shell` or Admin UI
3. **Environment Variables** (lowest) - Fallback only

For a GitOps-managed cluster, the config file approach makes the most sense. Credentials live in SOPS-encrypted Secrets, get decrypted by Flux, and mounted into the Filer automatically.

## The Implementation

### Step 1: Create the Credentials Config

I created a JSON config defining two users:

```json
{
  "identities": [
    {
      "name": "packer",
      "credentials": [
        {
          "accessKey": "packer",
          "secretKey": "<generated-secret>"
        }
      ],
      "actions": ["Admin", "Read", "Write", "List"]
    },
    {
      "name": "admin",
      "credentials": [
        {
          "accessKey": "admin",
          "secretKey": "<generated-secret>"
        }
      ],
      "actions": ["Admin"]
    }
  ]
}
```

### Step 2: SOPS Encryption

Initially, SOPS encrypted the *entire* Secret (every field), making git diffs useless. The solution: use `encrypted_regex` to only encrypt sensitive data:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.?secrets?\.ya?ml$
    encrypted_regex: ^(data|stringData)$
    age: age179hfp...
```

Now the Secret structure is visible in git, but `data` fields are encrypted:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: seaweedfs-s3-credentials
  namespace: seaweedfs
data:
  config.json: ENC[AES256_GCM,data:pXWU...]
```

### Step 3: Configure the Operator

The SeaweedFS Operator has built-in support for config-based authentication via `s3.configSecret`:

```yaml
filer:
  replicas: 1

  s3:
    enabled: true
    configSecret:
      name: seaweedfs-s3-credentials
      key: config.json

  iam: true  # Enable embedded IAM
```

The operator automatically:
- Mounts the Secret as `/etc/seaweedfs/config.json`
- Passes `-config=/etc/seaweedfs/config.json` to the S3 component
- Starts the IAM service on port 8111

### Step 4: Flux Decryption

Flux needed to know to decrypt the SOPS-encrypted Secret:

```yaml
# flux-kustomization.yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Without this, Flux tries to apply the encrypted blob directly to Kubernetes, which fails spectacularly.

## Testing

Unauthenticated requests now fail:

```bash
$ curl http://10.4.11.4:8333/
HTTP/1.1 403 Forbidden
Server: SeaweedFS 30GB 4.00
```

Authenticated requests work:

```bash
$ AWS_ACCESS_KEY_ID=packer \
  AWS_SECRET_ACCESS_KEY=... \
  aws s3 ls --endpoint-url http://s3.goldentooth.net:8333

$ AWS_ACCESS_KEY_ID=packer \
  AWS_SECRET_ACCESS_KEY=... \
  aws s3 mb s3://test-bucket --endpoint-url http://s3.goldentooth.net:8333
make_bucket: test-bucket
```

## What I Learned

**SOPS Encryption Modes**: SOPS has two modes - structured (encrypts individual YAML values) and binary (encrypts entire file). When piping through stdin, SOPS doesn't know the file type and defaults to binary. Using `--encrypt --in-place` after creating the file ensures structured encryption.

**encrypted_regex is Essential**: Without it, SOPS encrypts *everything* in Secrets, making git diffs show only that "something changed" without any context about what. With `encrypted_regex: ^(data|stringData)$`, you get clean diffs showing which keys changed while keeping values encrypted.

**Operator Config Mounting**: The SeaweedFS Operator's `s3.configSecret` abstraction is excellent. It handles all the volume mounting and argument passing automatically. Much cleaner than manually configuring volumes and args in a Deployment.

**IAM vs S3 Config Priority**: When using config file authentication (highest priority), it completely overrides filer-based configuration. There's no merging - it's an all-or-nothing hierarchy.