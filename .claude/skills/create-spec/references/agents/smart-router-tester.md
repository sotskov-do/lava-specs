# Smart-Router Tester (Phase 8 of create-spec)

You are a subagent dispatched by the create-spec orchestrator to boot the chain spec inside a dockerized **smart-router** and run a multi-node method probe. The orchestrator delegates this entire phase to you to keep its own context free of long-running boot/probe output.

Unlike the old local-provider flow, there is **no lava node, no gov proposal, no provider/consumer `screen` sessions, and no source build**. The smart-router is a prebuilt reverse-proxy image that loads the spec graph statically (`--use-static-spec`) and forwards requests to the chain's public RPC upstreams you configure. A boot of the router is seconds-to-a-minute, not 5–15 minutes.

**Execute the steps below verbatim. Do NOT improvise the config format and do NOT swap in a different image.** The image is `ghcr.io/magma-devs/smart-router:main` (see <https://github.com/Magma-Devs/smart-router/>).

## Inputs (substituted by orchestrator before dispatch)

- `<chain>` — the lowercased chain name (e.g., `iota`, `polygon`) — used for filenames/container name
- `<INDEX>` — the spec index UPPERCASE (e.g., `IOTA`, `IOTAT`) — must equal the `chain-id` in both the spec and the router config
- `<INTERFACE>` — one of `jsonrpc`, `rest`, `grpc`, `tendermintrpc`
- `<NODE_URL_1>`, `<NODE_URL_2>`, `<NODE_URL_3>` — 1–3 public node URLs (https://... ; ws/wss for subscriptions)
- `<WS_URL>` (optional) — a separate WebSocket URL for chains with subscriptions; if provided, add it to every `direct-rpc` block's `node-urls` list
- `<EXTRA_INTERFACES>` (optional) — for multi-interface chains (e.g., Cosmos: rest + tendermintrpc + grpc), a list of additional `(INTERFACE, urls)` blocks; each gets its own listener port (3361, 3362, …) and its own `direct-rpc` upstreams

## Optional read

For additional context only (the steps below take precedence over anything you find here): `references/phase4-testing-and-validation.md` (observe `END-OF-PHASE4-SENTINEL`).

## Step 0 — Resolve docker + authenticate to GHCR

The image is private on GitHub Container Registry. In CI, a `docker/login-action` step using `GITHUB_TOKEN` handles login before this phase runs. Locally, log in once with a token that carries the `read:packages` scope.

```bash
# Some hosts require sudo to reach the docker daemon socket. Detect once and
# reuse $DOCKER for every docker call below.
DOCKER="docker"; $DOCKER info >/dev/null 2>&1 || DOCKER="sudo docker"

IMAGE="ghcr.io/magma-devs/smart-router:main"

# Pull. If this fails with 401/403, you are not logged in to ghcr.io.
# Local one-time login (needs gh token with read:packages):
#   gh auth token | $DOCKER login ghcr.io -u "$(gh api user -q .login)" --password-stdin
# CI: handled by docker/login-action with ${{ github.actor }} + ${{ secrets.GITHUB_TOKEN }}.
$DOCKER pull "$IMAGE"
```

If the pull fails on auth, STOP and return `SMOKE: BOOT_FAILED` with the auth error — the orchestrator surfaces it to the user (who must `docker login ghcr.io`).

## Step 1 — Assemble the spec dir + write the router config

**1a. Spec dir.** `--use-static-spec` takes a *directory*; the router loads every `*.json` in it and resolves the candidate's `imports` from the same set. All specs in this repo live flat at the root, so copy them into a clean staging dir (this includes the candidate `<chain>.json` you are testing plus every parent it imports):

```bash
SPECS_DIR="/tmp/sr_specs_<chain>"
rm -rf "$SPECS_DIR" && mkdir -p "$SPECS_DIR"
cp ./*.json "$SPECS_DIR"/
ls "$SPECS_DIR"/<chain>.json   # candidate must be present
```

**1b. Router config.** Write `/tmp/sr_<chain>.yml`. The `endpoints[]` declare the local listeners; `direct-rpc[]` declare the upstream providers the router relays to. Use EXACTLY this shape (mirrors `config/smartrouter_examples/smartrouter_eth.yml` from the smart-router repo):

```yaml
metrics-listen-address: "0.0.0.0:7779"

endpoints:
  - listen-address: "0.0.0.0:3360"
    network-address: "0.0.0.0:3360"
    chain-id: "<INDEX>"
    api-interface: "<INTERFACE>"

direct-rpc:
  - name: "<chain>-upstream-1"
    chain-id: "<INDEX>"
    api-interface: "<INTERFACE>"
    node-urls:
      - url: "<NODE_URL_1>"
      # add `- url: "<WS_URL>"` here too if the chain has subscriptions
  - name: "<chain>-upstream-2"
    chain-id: "<INDEX>"
    api-interface: "<INTERFACE>"
    node-urls:
      - url: "<NODE_URL_2>"
  - name: "<chain>-upstream-3"
    chain-id: "<INDEX>"
    api-interface: "<INTERFACE>"
    node-urls:
      - url: "<NODE_URL_3>"
```

Rules — apply exactly:
- Create **one `direct-rpc` block per node URL** passed in (1, 2, or 3). Each is a distinct upstream the router can rotate across; this is what exercises the multi-node probe.
- `chain-id` is the spec index — UPPERCASE — and MUST match a `proposal.specs[].index` inside `<chain>.json`. A mismatch fails the router's startup spec resolution.
- **Subscriptions (HARD requirement, not optional).** If the spec enables ANY subscription method (`category.subscription: true` anywhere in `<chain>.json` — e.g. `eth_subscribe` inherited from ETH1), then **every** `direct-rpc` block's `node-urls` MUST include a `ws://`/`wss://` URL alongside the https URL. The router does NOT treat this as a warning: a provider with no ws upstream **fails verification and is excluded from the provider list**, and once every provider is excluded the router exits at boot with:
  ```
  static provider: failed creating chain router — excluding from provider list
  all static providers failed verification — cannot serve endpoint
  ```
  So: add `<WS_URL>` (or any `wss://` node URL) to every block. If the spec has subscriptions and you were given NO ws/wss URL, do not boot a doomed config — STOP and return `SMOKE: BOOT_FAILED` noting that a ws/wss upstream is required for this chain, so the orchestrator can ask the user for one. (Check with: `jq -e '[.proposal.specs[].api_collections[].apis[]?.category.subscription] | any' <chain>.json` → `true` means subscriptions are present.)
- **Multi-interface chains** (`<EXTRA_INTERFACES>` non-empty): add one more `endpoints[]` entry per extra interface on the next free port (`3361`, `3362`, …, same `chain-id`, the extra `api-interface`) AND a matching set of `direct-rpc[]` blocks for that interface's upstreams. See `config/smartrouter_examples/smartrouter_lava.yml` (rest + grpc + tendermintrpc) for the multi-interface shape.
- Do NOT add `add_on`/extension fields here — those live in the SPEC file (`collection_data.add_on`), not the router config.

After writing, confirm it parses as YAML and dump it:

```bash
cat /tmp/sr_<chain>.yml
python3 -c 'import yaml; yaml.safe_load(open("/tmp/sr_<chain>.yml"))' && echo "YAML OK"
```

## Step 2 — Boot the router container

Publish each listener port and mount the spec dir + config read-only. `--use-static-spec specs/` and the config path are resolved relative to the image WORKDIR `/smart-router`, so mount the spec dir at `/smart-router/specs` and the config at `/smart-router/sr.yml`.

```bash
$DOCKER rm -f sr_<chain> 2>/dev/null || true
$DOCKER run -d --name sr_<chain> \
  -p 3360:3360 \
  `# add -p 3361:3361 -p 3362:3362 ... one per extra-interface listener` \
  -p 7779:7779 \
  -v "$SPECS_DIR":/smart-router/specs:ro \
  -v /tmp/sr_<chain>.yml:/smart-router/sr.yml:ro \
  "$IMAGE" \
  sr.yml --geolocation 1 --use-static-spec specs/ --log-level debug --log-format json
```

## Step 3 — Wait for the router to be ready

The router verifies each upstream at startup (e.g. a `chain-id` check) before it begins serving. A spec-level defect — a bad `result_parsing`, a wrong `chain-id` `expected_value`, an unresolvable `imports` chain — surfaces here as a startup failure. That is the spec-level signal this phase exists to catch.

```bash
# Poll: ready when the listener answers, fail fast on fatal log lines.
timeout 120 bash -c '
  while true; do
    if curl -s -o /dev/null -m 3 http://localhost:3360 \
         -H "content-type: application/json" \
         -d "{\"jsonrpc\":\"2.0\",\"method\":\"\",\"params\":[],\"id\":1}"; then
      echo READY; break
    fi
    if '"$DOCKER"' logs sr_<chain> 2>&1 | grep -qiE "panic|fatal|failed to (load|expand|resolve) spec|verification failed|failed verification|cannot serve endpoint|no matching spec"; then
      echo FATAL; break
    fi
    sleep 2
  done'

$DOCKER logs --tail 60 sr_<chain> 2>&1
```

**Common boot failure — `all static providers failed verification — cannot serve endpoint`:** the most frequent cause is the missing-ws case above (the log will also say `websocket is not provided in 'supported' map`). Fix the config by adding a `wss://` upstream to every `direct-rpc` block and re-boot; do not treat it as a spec defect. Other causes: a wrong `chain-id` `expected_value` verification (the upstream's real chain-id differs) or an upstream that doesn't actually serve `<INTERFACE>` — those ARE spec/config issues to report.

For non-jsonrpc interfaces (`rest`/`grpc`/`tendermintrpc`) the readiness probe differs — for `rest`/`tendermintrpc` a `GET http://localhost:3360/<a known path>` returning any HTTP status means the listener is up; for `grpc` a TCP connect (`bash -c '</dev/tcp/localhost/3360'`) is sufficient. Use whichever matches `<INTERFACE>`.

If the logs show `panic`, `fatal`, `failed to load/expand/resolve spec`, `failed verification`, `cannot serve endpoint`, or `no matching spec` before the listener answers, STOP. Capture the full log (`$DOCKER logs sr_<chain>`), go to Step 6 (teardown), and return `SMOKE: BOOT_FAILED` with the excerpt (and, if it is the missing-ws case, say so explicitly). Do NOT skip to Step 4.

## Step 4 — Method-probe loop

**Before sending the first probe**, record the current log length so Step 4.5 can isolate only the lines emitted during probing:

```bash
P0=$($DOCKER logs sr_<chain> 2>&1 | wc -l)
echo "P0=$P0"
```

For every API in every collection of the current spec variant, send the call **through the router at `localhost:3360`** (the matching listener port for `<INTERFACE>`), NOT directly to the upstream `node-urls`. In production all traffic flows through the router, so this is the only representative path: it exercises client → router → upstream and surfaces spec-level parse/result-directive bugs that a direct-to-node call would hide.

| Category | Probe action |
|---|---|
| `category.stateful: 1` | SKIP. Record reason: "stateful — would broadcast transaction". |
| `category.subscription: true` | Open a WebSocket to the router (`ws://localhost:3360` — confirm the ws path in the startup logs), send the subscribe call with sample params from the spec's `parse_directive`/`block_parsing` hints, **wait up to 30 seconds for at least one message**, then send unsubscribe. PASS = ≥1 message received. FAIL = timeout. Probe via the router, not the upstream node. |
| Anything else | Build the simplest valid call from `block_parsing` + `parse_directive` hints and POST it to `http://localhost:3360` (jsonrpc/tendermintrpc) or GET the REST path (rest). Classify the router's response. |

Example jsonrpc probe:

```bash
curl -s -m 10 -X POST http://localhost:3360 \
  -H "content-type: application/json" \
  -d '{"jsonrpc":"2.0","method":"<method>","params":[],"id":1}'
```

Response classification:
- Response with `result` field (any value, including empty) → **PASS** (method exists and responded).
- Response with `error.code == -32601` → **FAIL** (method does not exist on chain / not routed).
- Response with `error.code == -32602` (invalid params) → **PASS-existence** (method exists; full functional probe would need correct args).
- Response with any other `error.code` → **WARN** (record code + message).
- Timeout (no response in 10s) → **TIMEOUT**.
- Node disagreement (different upstreams return materially different shapes for the same method) → **WARN-DISAGREEMENT** (record which upstream; the router rotates across them, so repeat the call a few times to surface disagreement).

## Step 4.5 — Scan the probe window for router errors

Step 4 only sees the router's HTTP/JSON-RPC reply. The router can return a plausible-looking body while logging a spec-level error the response classification misses (upstream rejection, result-parsing fallback, parse-directive mismatch). Scan the lines emitted **during the probe window** in the container log and fold real findings into the report.

The router runs with `--log-format json`, so each line is a JSON object with a `level` field. Slice from offset `P0`, keep only warn/error/fatal/panic levels, and drop known-benign patterns (best-effort allow-list — adjust if your run surfaces a new always-present noise line):

```bash
# benign noise to exclude (transient / environmental, NOT spec defects):
#   self signed certificate / x509     — public-endpoint TLS quirks
#   OTel / :4318 / metrics              — no local OTel collector
#   chain tracker / ChainTracker / UNKNOWN_BLOCK / DB Not Found / failed fetching data
#                                       — chaintracker probing archive-depth blocks (transient)
#   WebSocket SendRequest not implemented
#                                       — a wss upstream that only does subscriptions, not
#                                         request/response GET_BLOCKNUM; the router retries on
#                                         the paired http URL, so this is noise as long as an
#                                         http upstream is also present (it always should be)
BENIGN='self signed certificate|x509|OTel|:4318|otel|chain tracker|ChainTracker|WebSocket SendRequest not implemented|failed fetching data from the node|UNKNOWN_BLOCK|DB Not Found'

$DOCKER logs sr_<chain> 2>&1 | tail -n +$((P0+1)) \
  | grep -iE '"level":"(warn|error|fatal|panic)"|\b(WRN|ERR|FTL|PNC)\b' \
  | grep -Eiv "$BENIGN" || true
```

Router relay errors carry the RPC method inside a `method`/`request` field. Map them back to the probed method:

```bash
$DOCKER logs sr_<chain> 2>&1 | tail -n +$((P0+1)) \
  | grep -iE '"level":"(error|fatal|panic)"' | grep -Eiv "$BENIGN" \
  | grep -oE '"method":"[a-zA-Z0-9_]+"' | sort | uniq -c
```

For each surviving line:
- **Method-associated** (has a `method=`/`request=` field): downgrade that method's classification to at least **WARN** in the report, citing the error text and any code. If Step 4 already classified it FAIL on the same `-32601`, keep FAIL — do not double-count.
- **Not method-associated** (router-wide error): record it in the report's "Log-scan findings" section.
- **`fatal`/`panic` after a successful boot**: a runtime crash *during probing*. Capture the full line, flag it prominently, and note the run is unreliable.

## Step 5 — Write the probe report

```bash
mkdir -p docs/<chain>
```

Write `docs/<chain>/METHOD_PROBE_REPORT.md`:

```markdown
# Method Probe Report — <chain>

Generated: <UTC timestamp>
Router image: ghcr.io/magma-devs/smart-router:main
Router config: /tmp/sr_<chain>.yml
Spec variant: <INDEX> (<INTERFACE>)
Upstreams probed: <URL_1>, <URL_2>, <URL_3>

| Method | Classification | Upstream notes | Notes |
|---|---|---|---|
| <method> | <PASS/FAIL/SKIP/WARN/TIMEOUT> | <code(s)> | <one-line note> |
| ... |

## Log-scan findings (probe window)

Non-benign warn/error/fatal/panic lines logged during probing (Step 4.5). Empty table = clean.

| Level | Associated method | error excerpt |
|---|---|---|
| ERR | <method or —> | <excerpt> |
```

## Step 6 — Tear down

```bash
$DOCKER rm -f sr_<chain> 2>/dev/null || true
```

Repeat Steps 1–6 for each `(spec_variant, api_interface)` pair if more than one was passed in. Each iteration uses a fresh container (no shared state between runs).

## Return to orchestrator

Return a short summary:

1. The path to `docs/<chain>/METHOD_PROBE_REPORT.md`.
2. Counts: `PASS=<n> FAIL=<n> SKIP=<n> WARN=<n> TIMEOUT=<n> LOG_WARN=<n>` (`LOG_WARN` = non-benign log-scan lines from Step 4.5).
3. The names of any FAIL/TIMEOUT methods (one per line), plus any method downgraded to WARN by the log scan, so the orchestrator can decide whether to fix the spec before Phase 9.
4. Teardown status (container removed / leftover container).

If the router could not boot, return `SMOKE: BOOT_FAILED` plus the relevant `$DOCKER logs` excerpt instead.

Do NOT echo the full probe report into your response — the orchestrator reads it from disk if it needs the detail.

END-OF-SMART-ROUTER-TESTER-SENTINEL
