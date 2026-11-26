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

## Tailscale Funnel

- `tailscale.com/funnel: "true"` only works on a standalone Tailscale Ingress. Do **not** combine it with `tailscale.com/proxy-group`: the operator's ProxyGroup ingress reconciler ignores the funnel annotation, so the Ingress stays tailnet-only.
- Funnel Ingresses (e.g. `apps/uptime-kuma/ingress.yaml`, `apps/executor/ingress.yaml`) need only `ingressClassName: tailscale`, the funnel annotation, and a `tls.hosts` entry.
- The tailnet ACL must grant the `funnel` node attribute to `tag:k8s` (`autogroup:member` doesn't cover tagged devices).
- **Migrating an Ingress off a ProxyGroup** (removing `tailscale.com/proxy-group`) leaves stale state that breaks the new standalone proxy. Before/after the switch, clean up:
  1. Delete the leftover cert secret `<host>.tail5158.ts.net` in the `tailscale` namespace. It keeps the `tailscale.com/parent-resource=<name>` labels, so the operator counts it as a child of the new proxy and fails with `failed to create or get API key secret: input does not match format`. Delete it by name, not by label.
  2. Delete the stale `svc:<host>` Tailscale Service in the admin console (Services page). The operator skips its cleanup once the ProxyGroup annotation is gone. Until it's removed, the new proxy registers as `<host>-1`.
  3. If the proxy already registered under the wrong name, delete its state secret `ts-<name>-*-0` and pod so it re-registers, then remove the old offline device in the admin console.
