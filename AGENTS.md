# AGENTS.md — guidance for AI agents (and humans) working in this repo

## This repo is GitOps-managed (Flux)

The homelab cluster is reconciled by **Flux** from `origin/main` (the `apps` and
`flux-system` Kustomizations in `flux-system`). Consequences:

- **Manual `kubectl apply/edit/create` does not persist.** Flux detects drift on its
  reconciliation interval and reverts live objects back to the state committed to
  `origin/main`. A change applied from a local, unpushed checkout will be undone
  within minutes.
- **To persist ANY change:** commit it, push to `origin/main`
  (`git push origin main`), and let Flux reconcile it. Verify afterward that the
  live state matches (Flux status: `kubectl -n flux-system get kustomization`).
- **Editing live objects to test things is fine for immediate debugging**, but
  expect Flux to fight you; make the same edit in the repo, or use a short-lived
  object outside the Kustomizations' scope.
- **Pods are never edited live.** Image updates and config changes arrive through
  commits (many via Renovate PRs), Flux applies them, and the controllers
  (helm/daemonset operators, kubelet) roll the workloads.

## Cluster access notes

- Default kubeconfig context `admin@homelab` talks directly to the node; the old
  operator-FQDN context is dead. `talosctl` needs `TALOSCONFIG` set and
  `GODEBUG=rsa1024=1`.
- Secrets are committed nowhere; there is no SOPS/sealed-secrets. Anything that
  must exist as a Kubernetes Secret (Flux cannot recreate these on rebuild) is
  created out-of-band — check `docs/` first if you are about to reference one.

## Repo conventions

- Raw kustomize manifests under `apps/<app>/`, wired up in the app's
  `kustomization.yaml` (and `clusters/homelab/` for the cluster-level ones).
  Image tags/digests are updated by Renovate — keep pins digest-pinned.
- `docs/` holds design/decision notes worth keeping (e.g. `docs/node-tailscale-gateway-spec.md`,
  `docs/dns-strategy.md`). When you solve something non-obvious on live infra,
  write the recipe there.
- Grafana dashboards live in `apps/observability/dashboards/` as JSON and are
  imported manually into Grafana (not provisioned by Flux).