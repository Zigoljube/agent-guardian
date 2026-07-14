# Agent Blackbox — fresh install gets 0 curated / 0 community threats: staging curator unreachable, public tier not served by any peer

**Reporter:** Zigoljube (fresh clean install, Apple Silicon)
**Blackbox version:** `feat/guardian` @ `90f23c687`; DKG v10.0.6 built from `matic031/dkg#feat/blackbox` @ `a7ba768`; network `mainnet-base`, `nodeRole: edge`
**Severity:** blocks the core value proposition — a fresh install ends with an empty ruleset and no protection, and `hermes blackbox sync --wait --require-rules` can never succeed

## Summary

A clean install of Agent Blackbox (installer completes, node bootstraps, Blazegraph store healthy, 14–15 network peers connected) ends with **zero curated public threats and zero community threats**, permanently. Two independent upstream causes, plus one installer bug on Apple Silicon:

1. **The curator for `umanitek/blackbox-threats-staging` is unreachable.** Exactly one peer on the network identifies as curator (`12D3KooWBY9jmNATMPv1DZcKbFas5RtjpkhT69pPwvkUBY2MMnDX`); it is not dialable (relay `NO_RESERVATION`, then timeouts). Every `request-join` returns 502 `"No reachable curator found"`. Without admission, the daemon (correctly) skips SWM sync — so the **community tier can never load**.
2. **No reachable peer serves the public (verifiable-memory) tier either.** Catch-up connects to all discoverable peers and returns `data 0, shared memory 0`, status `unreachable` — for the new graph *and* for the previous `umanitek/guardian-threats-staging` (which served `data 4371` from 6 peers on 2026-07-10). The data-serving node(s) went dark between Jul 10 and Jul 14. Curated assets appear to be hosted only by Umanitek-operated node(s), so when they're offline the public tier is gone for everyone who hasn't already synced.

Net effect: the installer's own success gate (`hermes blackbox sync --wait --require-rules`) is unsatisfiable, and the dashboard shows an empty threat graph.

## Environment

- macOS (Apple Silicon, arm64), Node.js v25.5.0, Python 3.11
- Fresh install via `scripts/blackbox-install.sh` (no prior state; previous ~16 GB node home wiped)
- Store: Blazegraph (operator-managed, native jar — see the Apple Silicon note below), quads namespace `agent-blackbox`
- Node healthy: `/api/status` 200, store SPARQL answers, `peers 14/14 (connected 15)`

## Reproduction

1. Clean machine (or wipe `~/agent-guardian/.dkg` + `~/.hermes` blackbox state).
2. `bash scripts/blackbox-install.sh` — completes; node bootstraps on `mainnet-base`.
3. `hermes blackbox sync --wait --require-rules`

Result:

```
warning: could not request join for umanitek/blackbox-threats-staging:
POST /api/context-graph/umanitek%2Fblackbox-threats-staging/request-join -> 502:
{"error":"Could not deliver join request to curator. No reachable curator found.",
 "errors":["BY2MMnDX: dial failed (substrate queued (transport): Network identity probe
   failed for 12D3KooWBY9jmNATMPv1DZcKbFas5RtjpkhT69pPwvkUBY2MMnDX: failed to connect
   via relay with status NO_RESERVATION)",
   "K92Xapyy: not curator","5phz8m91: not curator","y1s97WVs: not curator", ...]}
Ruleset synced from umanitek/blackbox-threats-staging:
  0 injection, 0 escalation, 0 dependency
  0 rules — no local public/SWM threat rows are available yet.
Required ruleset sync failed; install is incomplete until threat rows load from DKG.
```

Daemon log (loops every ~60s):

```
[DKGAgent] Skipping SWM sync for unauthorized or unconfirmed context graph "umanitek/blackbox-threats-staging"
[DKGAgent] Catch-up sync for "umanitek/blackbox-threats-staging": peers=1/1 data=0 sharedMemory=0
[DKGAgent] Connected catchup peer admission probe failed for 5T3m5Kr3: Network identity probe fail...
```

Direct curator probe:

```
$ dkg peer info 12D3KooWBY9jmNATMPv1DZcKbFas5RtjpkhT69pPwvkUBY2MMnDX
Connected:     no
Connections:   0
Sync Capable:  no
Sync Status:   unknown (last success: never)
```

Public-tier catch-up on the *previous* graph confirms the data source is gone, not the client:

```
$ dkg subscribe umanitek/guardian-threats-staging --save     # succeeds
$ dkg sync catchup-status umanitek/guardian-threats-staging
Status:  unreachable
Result:  peers 14/14 (connected 15), data 0, shared memory 0
# same command on 2026-07-10 (pre-rename): peers 6/6, data 4371
```

## Expected

- A fresh install either loads the curated ruleset, or fails with an actionable message pointing at the curator/graph availability — and recovers automatically once the curator returns (the 60s retry loop does this part correctly).
- The curated **public** tier should survive the curator being offline: verifiable-memory assets are on-chain-anchored and could be replicated/served by core nodes rather than only by the curator's own infrastructure.

## Suggested fixes

1. **Bring the staging curator back / keep it highly available**, or document its multiaddr so operators can peer with it explicitly.
2. **Replicate curated VM assets** to (or pin them on) core mainnet nodes so public-tier catch-up doesn't depend on a single operator node being online.
3. **Surface the state in `hermes blackbox status` and the dashboard** — currently they just show `ruleset: 0 ...`, which reads as "no threats exist" rather than "curator unreachable, admission pending". A one-line `graph: waiting for curator admission (last attempt: <ts>, <err>)` would save every operator this whole investigation.
4. Consider shipping a **bundled baseline ruleset** (the repo already has the 20k seed bundle tooling) as an offline fallback so audit-mode protection isn't zero while the graph is unreachable.

## Related: installer cannot provision Blazegraph on Apple Silicon (amd64-only image)

`scripts/blackbox-install.sh` → `provisionBlazegraphDocker` pins `lyrasis/blazegraph:2.1.5`, which has **no arm64 build**. On an Apple Silicon Mac (Colima/Docker), the container runs under QEMU x86_64 emulation: the JVM burned 19+ minutes of CPU without ever binding port 8080, so the 120s readiness probe (`/bigdata/status`) always times out:

```
Starting Blazegraph container "dkg-blazegraph-agent-blackbox" on port 9999…
Blazegraph setup failed: Blazegraph did not become ready within 120000ms at
http://127.0.0.1:9999/bigdata/status. Container started but the SPARQL endpoint is not responding.
```

**Workaround that works** (and could be automated as an installer fallback): run Blazegraph natively — arm64 JDK + the official `blazegraph.jar` 2.1.6 starts in ~8 s (runs fine even on Java 26), create the quads namespace from `BLAZEGRAPH_NAMESPACE_XML_TEMPLATE`, then install with the operator-managed store path:

```
BLACKBOX_DKG_STORE_URL="http://127.0.0.1:9999/blazegraph/namespace/agent-blackbox/sparql" \
  bash scripts/blackbox-install.sh
```

Suggested fix: use/publish a multi-arch Blazegraph image, or detect `arm64` + amd64-only image in the provisioner and fall back to the native-jar path (or at least raise the readiness timeout and warn about emulation).
