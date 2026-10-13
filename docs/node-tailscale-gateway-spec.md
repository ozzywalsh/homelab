# Spec: node-level Tailscale + Traefik gateway (the "endgame" remote access shape)

Status: DRAFT — for implementation by a future session.
Inspired-by: /Users/ozzywalsh/dev/hmlb (klimlive): Talos `siderolabs/tailscale`
extension per node + hostNetwork Envoy gateway + tail-IPs published in public
DNS. This spec adapts that to this repo's stack: Traefik instead of Envoy,
same DNS philosophy, same cert philosophy.

## 1. Goal

Every app is reachable as `https://<app>.wlsh.dev` from:

| client location | resolves to | path |
|---|---|---|
| LAN, no Tailscale (TV) | node LAN IP (local DNS override) | plain LAN |
| LAN or remote, Tailscale on | node tailnet IP (`100.x`) | WireGuard, single NAT, direct |
| remote, no Tailscale | nothing reachable | fail-closed by construction |

No router port forwards. Home public IP never appears in DNS. No per-app
Tailscale identities.

## 2. Current state this builds on (verified 2026-10-13)

- Traefik: `apps/traefik/helm-traefik.yaml` — chart 41.0.2, 2 replicas,
  LoadBalancer 192.168.1.200 via MetalLB, entrypoints web(80)/websecure(443).
- cert-manager: `apps/cert-manager/helm-cert-manager.yaml` — installed, CRDs
  on, **no ClusterIssuer exists yet**.
- Cloudflare: zone for `wlsh.dev`; `apps/ddns` already holds a CF API
  token in a secret (reuse or issue a new scoped token; do not reuse the
  DDNS token as-is — least privilege).
- Tailscale: `apps/tailscale/` — Operator + ingress/egress ProxyGroups live.
  `apps/qbittorrent/jellyfin-tailscale.yaml` — bespoke hostNetwork proxy
  deployment + serve config (the current fix). **Decommissioned at cutover
  (step 6).**
- Talos machine configs: `infra/clusterconfig/{controlplane,worker}.yaml`
  (Omni-era templates; verify actual application method before editing —
  `talosctl apply-config` vs Omni patches).
- Jellyfin: `apps/qbittorrent/jellyfin.yaml` — pod netns, PVCs on
  `local-path` (node-pinned). Keep in pod netns throughout.

## 3. Architecture

    tailnet client ──WireGuard──> node tailscale0 (100.x, direct via single NAT)
                                  └──> Traefik (hostNetwork, :443 on node)
                                        └──IngressRoute──> app Service
    LAN client    ──────────────> node LAN IP :443 ──> same Traefik

DNS (`*.wlsh.dev`, CF): A records -> node tailnet IPs (multiple records
= round-robin across gateway nodes). Certs: cert-manager DNS-01 wildcard —
issuance never depends on where records point.

## 4. Implementation steps

### 4.1 Talos: tailscale extension on all nodes

In `infra/clusterconfig/controlplane.yaml` + `worker.yaml` (or the equivalent
Omni patch source — CONFIRM which file actually drives node config):

    extensionServices:
      - name: tailscale
        environment:
          - TS_AUTHKEY=${ts_auth_key}
          - TS_EXTRA_ARGS=--advertise-tags=tag:k8s

- `siderolabs/tailscale` extension must be present in the image schematic;
  if nodes are factory-built images, rebuild schematic / re-image only worker
  nodes first (control planes can follow later — gateway lives on one
  worker... see 4.3).
- Auth key: pre-auth key or OAuth in `infra/secrets.yaml` (sops). Key must
  NOT expire before the next node re-provision; note expiry in the file.
- Verify after apply, from the node: `talosctl services` shows tailscale
  running; admin console shows the node device.

### 4.2 cert-manager: DNS-01 wildcard issuer

New file `apps/cert-manager/issuer-letsencrypt-dns01.yaml`:

- `ClusterIssuer letsencrypt-prod`: ACME server LE prod, solver `cloudflare`
  referencing secret `cloudflare-api-token` (new, scoped: `Zone.DNS Edit`
  for `wlsh.dev` only).
- `Certificate wildcard-ozzywalsh-com` (ns cert-manager):
  dnsNames `wlsh.dev` + `*.wlsh.dev`, secretName
  `wildcard-ozzywalsh-com-tls`, renewBefore 720h.
- Feed the wildcard to Traefik's default cert via a `TLSStore`
  (`default` in traefik ns) so `websecure` serves it without per-route
  secretRefs.

### 4.3 Traefik: become the mesh-facing listener

Constraint discovered up front (write this into the PR): MetalLB VIP
`192.168.1.200` routes to pods via kube-proxy; tailnet clients CANNOT reach
the VIP. The listener must bind on the node host. Change, in
`apps/traefik/helm-traefik.yaml`:

- `deployment:` -> `daemonSet:` (chart supports kind: daemonSet).
- `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet` (values block).
- `nodeSelector`: the node currently hosting jellyfin's PVCs (local-path
  pins jellyfin there; co-locating the gateway keeps the hairpin local).
  Label the node in Talos node labels or via kubectl; note hostname.
- Ports: set `websecure.port: 443` (hostNetwork binds the real port; keep
  `web` on 80). The LoadBalancer svc keeps working for LAN clients that use
  192.168.1.200 — it now DNATs to hostNetwork pods, verify `externalTrafficPolicy:
  Local` still applies.
- Keep `replicas`-equivalent: DaemonSet on ONE gateway node = 1 listener;
  future gateway nodes are added by widening the nodeSelector and adding an
  A record each.

### 4.4 DNS

- Public zone (Cloudflare): `A jellyfin.wlsh.dev -> <node 100.x>`
  (+ one record per gateway node later). TTL 300, proxied: false.
- Local override (OPEN DECISION below): wildcard `*.wlsh.dev` ->
  gateway node LAN IP via home resolver (AdGuard/unbound/pfSense) so
  TV/lan devices without Tailscale work. Until that exists, LAN-only
  clients use `jellyfin.homelab` via the existing Traefik IngressRoutes.
- ddns (`apps/ddns`): keep for now; OPEN DECISION to retire.

### 4.5 Migration order (downtime ~1 min, jellyfin only)

1. Apply 4.1 (nodes join tailnet), capture node tail IPs.
2. Apply 4.2, confirm wildcard cert in secret.
3. Apply 4.3 (Traefik DaemonSet hostNetwork). Verify: from LAN,
   `curl -k https://<node-lan-ip>` serves Traefik's cert/dashboard;
   ingressRoutes still answer.
4. DNS 4.4: publish A record -> tail IP; test from a remote machine with
   Tailscale ON: `https://jellyfin.wlsh.dev` streams.
5. (Optional, after stable 48h) step 6.
6. Decommission: delete `apps/qbittorrent/jellyfin-tailscale.yaml`, remove
   from kustomization, scale-delete its pods; remove
   `jellyfin-tailscale-serve` refs. Keep `apps/tailscale` operator +
   ProxyGroups (ingress ProxyGroup still unused; egress serves
   `manderley-proxy` -> tailnet fqdn needs).

### 4.6 Verification checklist

- `tailscale ping <node>` from Mac: `direct`, not DERP (single NAT now).
- From phone (tailnet, LTE): jellyfin streams; `sonarr.wlsh.dev`
  (once routed) works.
- From TV (LAN, no tailscale): `jellyfin.wlsh.dev` via local override.
- From a non-mesh remote client: connection timeout, not an error page.
- No `0.0.0.0` port forwards exist on the router for 80/443.

## 5. Friends access (jellyfin only)

- Primary: **machine sharing** — share the gateway node in Tailscale admin
  to friends' personal accounts. Their `jellyfin.wlsh.dev` resolves
  (public record) and routes because the node is shared. ACL rows allow
  only port 443 on that node. Auth is Jellyfin's own accounts.
- Do NOT invite friends as tailnet users (seat limits, blast radius).
- OPEN fallback (not in scope): Cloudflare Tunnel for a public tier.

## 6. Open decisions (resolve before/after implement, listed explicitly)

1. Provisioning source for machine config (talosctl vs Omni) — blocks 4.1.
2. Local DNS override mechanism for LAN clients (4.4).
3. Retire `apps/ddns` or keep for misc records.
4. Traefik MetalLB VIP destiny (keep for LAN-legacy or drop).
5. Auth posture for arr-stack hostnames (`oauth2-proxy`+authentik like
   klimlive, or keep tailscale-only reachability).

## 7. Non-goals

- Replacing `jellyfin.homelab` internal routes.
- Gateway API / HTTPRoute migration (Traefik v3 supports it; revisit only
  if a second-tier Envoy ever becomes justified).
- Mastodon-style fully public tiers.

## 8. Rollback

Each step is independently revertible: 4.1 remove extensionServices; 4.2
delete issuer/cert; 4.3 revert helm values (MetalLB VIP path returns); DNS
A records point back at nothing (delete). The jellyfin bespoke tailscale
deployment stays until step 6, so tailnet access via the old path exists
throughout 4.1-4.5.
## 9. Implementation status (2026-10-13)

- **Domain**: user confirmed `wlsh.dev` (NOT ozzywalsh.com).
- **Talos**: `tailscale` extension v1.102.2 IS in the node image
  (`talosctl get extensions` / `ext-tailscale` service state "Waiting for
  extension service config" since boot). No re-image needed — only an
  `extensionServices` machine-config entry with `TS_AUTHKEY` is missing.
  Applied against single node `talos-c24-t0f` (192.168.1.146); talosconfig
  in `infra/clusterconfig/talosconfig` has empty endpoints — synthesize one
  (endpoints: [192.168.1.146]) or use the working copy at /tmp pattern.
  Server TLS uses a small RSA key: client needs `GODEBUG=rsa1024=1`.
- **Traefik chart gotchas** (41.0.2): `hostNetwork` is a TOP-LEVEL chart
  value, but `dnsPolicy` lives under `deployment:`; with hostNetwork,
  `updateStrategy.rollingUpdate.maxUnavailable` (top-level, also under
  `updateStrategy`) must be > 0. Preflight via `helm template traefik
  <chart> -f <values>` catches all of these.
- **Gateway node** = `talos-c24-t0f` (single-node cluster; everything
  co-located).
- **Implementation note**: `apps/qbittorrent/jellyfin-tailscale.yaml` was
  the hostNetwork sidecar-pod gateway; the node extension now replaces it
  (two tailscaled in one netns cannot coexist — pod must be deleted when
  the extension goes live; flux prune handles it once the file is removed,
  which is done in this cutover).
- **Identity change**: the old `jellyfin` tailnet device (100.121.228.84)
  disappears with the pod; the node registers as a NEW device (hostname
  gateway talos-c24-t0f). Any tailnet references to the old fqdn break at
  cutover; new device IP is what DNS points at.
- **DNS**: pending — user creates `A jellyfin.wlsh.dev -> <node tail IP>`
  in Cloudflare (or provides a scoped token; ddns token unusable, its
  secret is missing and pods are in CreateContainerConfigError).
