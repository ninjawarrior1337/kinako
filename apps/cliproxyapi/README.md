# cliproxyapi

[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) fronting **OpenCode Go**
(`https://opencode.ai/zen/go`) so any OpenAI/Anthropic-compatible client can use the
Go model catalog through one endpoint:

```
https://cliproxyapi.tail5158.ts.net        # tailscale ingress
http://cliproxyapi.apps.svc.cluster.local:8317   # in-cluster
```

| File | Notes |
|---|---|
| `deployment.yaml` | image + volume mounts |
| `secret.sec.yaml` | **plaintext config** — local only, gitignored |
| `secret.yaml` | SOPS-encrypted config — this is what is committed |
| `service.yaml` / `ingress.yaml` | ClusterIP :8317 + tailscale ingress |
| `pvc.yaml` | `cliproxyapi-data` (`/root/.cli-proxy-api`, `plugins/`) |

The whole CLIProxyAPI config lives in `secret.sec.yaml` under `stringData.config.yaml`
and is mounted at `/CLIProxyAPI/config.yaml`. It is mounted from a Secret, so the
deployment carries `secret.reloader.stakater.com/reload: "cliproxyapi-config"` —
the pod restarts automatically when the Secret changes.

## The three protocol surfaces

OpenCode Go exposes each model on **one specific endpoint**, and the model must be
listed under the matching provider block or requests fail upstream:

| OpenCode Go endpoint | CLIProxyAPI block | Client-side API |
|---|---|---|
| `https://opencode.ai/zen/go/v1/chat/completions` | `openai-compatibility` | `POST /v1/chat/completions` |
| `https://opencode.ai/zen/go/v1/responses` | `codex-api-key` | `POST /v1/responses` |
| `https://opencode.ai/zen/go/v1/messages` | `claude-api-key` | `POST /v1/messages` |

Every block shares the same OpenCode Go API key and forwards the client session to
OpenCode Go as `x-opencode-session: "$CPA-SESSION-ID"` (OpenCode Go requires a
session id; `$CPA-SESSION-ID` is CLIProxyAPI's internal session id and is omitted
when none is found). The `claude-api-key` block additionally sends `x-api-key:`
because CLIProxyAPI only sends `Authorization: Bearer` to non-Anthropic base URLs
while OpenCode Go's `/messages` authenticates with `x-api-key`.

Currently registered (25 models):

- **`openai-compatibility`** — glm-5.3-flash, glm-5.3, glm-5.2, kimi-k3,
  kimi-k2.7-code, longcat-2.0, mimo-v2.6-flash, mimo-v2.6-pro, mimo-v2.5,
  mimo-v2.5-pro, minimax-m3, qwen3.8-max, qwen3.8-flash, qwen3.7-plus,
  deepseek-v4.1-flash, deepseek-v4-pro, deepseek-v4-flash,
  deepseek-v4-flash-vision-exp, hy4-preview, hy3
- **`codex-api-key`** — gpt-6-luna, gpt-5.6-luna, muse-spark-1.3-contributor,
  muse-spark-1.2-contributor
- **`claude-api-key`** — minimax-m2.7

## Adding or updating models

### 1. Get the current catalog and its endpoints

Fetch <https://opencode.ai/docs/go/> and read the **Endpoints** table. It lists, per
model, the model id and the endpoint it is served on. Map that endpoint to a block
using the table above.

> Skip the **temporary / "limited time"** models (`longcat-2.5-preview-free`,
> `space-bunny-free`). Everything else listed on that page belongs in the config.

### 2. Find what is stale

```bash
# what the proxy currently serves
curl -s https://cliproxyapi.tail5158.ts.net/v1/models \
  | grep -o '"id":"[^"]*"' | cut -d'"' -f4 | sort

# what the config registers
awk '/^              - name: "/{m=$0; sub(/^ *- name: "/,"",m); sub(/".*/,"",m); print m}' \
  apps/cliproxyapi/secret.sec.yaml | sort
```

Anything in the docs but not in `/v1/models` is missing — requesting it returns
`400 unknown provider for model <id>`.

### 3. Edit the plaintext config

Add the model to the `models:` list of the **correct block** in
`apps/cliproxyapi/secret.sec.yaml`. `name` is what the client sends, `alias` what
OpenCode Go receives, `display-name` is cosmetic; for OpenCode Go all three are the
same model id.

```yaml
            models:
              - name: "kimi-k2.6"
                alias: "kimi-k2.6"
                display-name: "Kimi K2.6"
```

Keep the 8-space indentation — this is all inside the `config.yaml: |` block scalar.

### 4. Encrypt

Per `AGENTS.md`, only the encrypted file is committed:

```bash
cd <repo root>
export SOPS_AGE_KEY_FILE=./age.agekey
sops --encrypt --age age1w67tp7vumngt0358xh63ayhlwkqc04tc7qk0nu6a7weg6arlqyxsrrexy6 \
  apps/cliproxyapi/secret.sec.yaml > apps/cliproxyapi/secret.yaml
```

Sanity check the round-trip before committing:

```bash
sops --decrypt apps/cliproxyapi/secret.yaml | diff - apps/cliproxyapi/secret.sec.yaml \
  && echo "round-trip identical"
```

### 5. Commit and push

Flux syncs the new Secret, the reloader annotation rolls the pod, and the new model
is live. Verify:

```bash
# model is advertised
curl -s https://cliproxyapi.tail5158.ts.net/v1/models | grep -o '"id":"kimi-k2.6"'

# and answers (use the surface matching its block)
curl -s https://cliproxyapi.tail5158.ts.net/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"kimi-k2.6","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

To confirm Flux actually picked it up:

```bash
kubectl get pods -n apps -l app=cliproxyapi          # AGE should be seconds
```

## Testing changes without touching the cluster

The config-driven behaviour (protocol mapping, `payload.*` rules) can be checked
against a throwaway instance using the same image:

```bash
mkdir -p /tmp/cpa && cp apps/cliproxyapi/secret.sec.yaml /tmp/cpa/x.yaml
# extract stringData.config.yaml from x.yaml into /tmp/cpa/config.yaml, put the real
# api-key in it, change `port:` to something free, then:
docker run -d --name cpatest -p 18317:18317 \
  -v /tmp/cpa/config.yaml:/CLIProxyAPI/config.yaml:ro \
  docker.io/eceasy/cli-proxy-api:v7.3.20
docker logs -f cpatest        # confirms the YAML parses
```

Importantly, this catches YAML the real parser rejects but a naïve reader accepts.

Testing OpenCode Go directly (bypassing the proxy) also works, but you **must** send a
session header or upstream rejects you with a misleading error:

```bash
KEY=$(grep -o 'sk-[A-Za-z0-9_-]*' apps/cliproxyapi/secret.sec.yaml | head -1)
curl -s https://opencode.ai/zen/go/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -H "x-opencode-session: $(uuidgen)" \
  -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-v4.1-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

Without `x-opencode-session` you get `400 MissingSessionID`. Note the api-key values
in the YAML are quoted — strip the quotes if you extract them by hand, or every call
comes back `401 Invalid API key`.

## Known issues

- **Grok 4.7 / 4.6 are intentionally removed** (`b48757c`). OpenCode Go's Grok backend
  rejects two Codex-specific tool shapes that every other backend accepts, failing with
  opaque errors instead of a useful message:
  - `tools[].type == "namespace"` (Codex's grouped MCP/sub-agent tools) →
    `422 Upstream request failed: Endpoint is unavailable.`
  - `web_search.external_web_access` / `search_content_types` →
    `400 Argument not supported: external_web_access`

  To re-add them, also add a `payload.filter` for the Grok models (see commit
  `5ebe4f2`) using `tools.#(type=="namespace")#` — the trailing `#` is required to
  match *all* array entries; without it nothing is removed. Be aware that Grok then
  loses the grouped MCP/sub-agent tools.

- **`kimi-k2.6`** is listed on the OpenCode Go docs page but is not registered here.
  Add it to `openai-compatibility` if you want it.

- The docs' **Endpoints** column is the recommended surface, not a hard rule: several
  models documented under `/v1/messages` (e.g. `qwen3.8-max`, `minimax-m3`) also serve
  fine on `/v1/chat/completions` and are registered there. If a model returns
  `ModelProtocolUnsupported`, move it to the block matching its documented endpoint.

- Re-encrypting `secret.yaml` rewrites the encrypted data key, so the git diff looks
  like a whole-file change even for a one-line edit. That is normal — read the diff
  with `sops --decrypt` instead.
