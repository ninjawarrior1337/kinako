# AGENTS.md

## Secrets workflow

Secrets follow a two-file pattern:

| File | Purpose | In git? | Content |
|---|---|---|---|
| `*.sec.yaml` | Local plaintext working copy | No (gitignored) | Unencrypted values |
| `*.yaml` | Committed encrypted secret | Yes | SOPS-encrypted with age key |

### Flow

1. **Edit** the `.sec.yaml` file with real secret values (plaintext).

```
path/to/secret.sec.yaml    ← unencrypted, local only
```

2. **Encrypt** it into the committed `.yaml` file:

```bash
export SOPS_AGE_KEY_FILE=./age.agekey
sops --encrypt --age <recipient> path/to/secret.sec.yaml > path/to/secret.yaml
```

3. **Commit** the encrypted `.yaml` file. Flux syncs it and decrypts on the cluster automatically.

```
path/to/secret.yaml        ← SOPS-encrypted, in git
```

### Never

- Commit `.sec.yaml` files (they're in `.gitignore`)
- `kubectl apply` secrets manually — Flux handles decryption and deployment
