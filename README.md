# ozlabs-fleet

The GitOps repository for the OzLabs lab cluster: a single-node K3s cluster reconciled by
[Flux](https://fluxcd.io). The baseline cluster runs Flux and nothing else. Every workload is an
**opt-in target**, switched on or off by one key in a ConfigMap.

**No secrets in this repo, ever.** It is public, the cluster reads it without credentials, and
nothing in it may need one: no tokens, keys, passwords or private URLs.

## Layout

```
clusters/ozlabs-k3s/
  flux-system/
    gotk-components.yaml   Flux controllers (exported, committed unchanged)
    gotk-sync.yaml         GitRepository + Kustomization "flux-system" for this repo
    kustomization.yaml     both files + the kustomize-controller patch (below)
  kustomization.yaml       flux-system + targets.yaml
  targets.yaml             Kustomization "targets": builds ./targets with the switch values
targets/
  kustomization.yaml       one line per target switch
  online-boutique/
    switch.yaml            Kustomization "online-boutique": path ./on or ./off
    off/                   renders nothing
    on/                    namespace, pinned app source, app Kustomization
VERSION                    pinned Flux version
```

## Pinned versions

| Component | Version |
|---|---|
| Flux | **v2.9.6** (`VERSION`) |
| source-controller | v1.9.6 |
| kustomize-controller | v1.9.6 |

Only `source-controller` and `kustomize-controller` are installed: Flux pulls from public Git and
applies Kustomizations. Nothing listens for webhooks, so nothing in the cluster needs inbound access.
`helm-controller`, if a target ever needs it, is added by a commit here.

## Target switches: the `ozlabs-targets` ConfigMap

| Item | Contract |
|---|---|
| ConfigMap | `ozlabs-targets` in namespace `flux-system` |
| Switch key | `TARGET_<NAME>` (target name in upper snake case), value `on` or `off` |
| Sub-keys | `TARGET_<NAME>_<KEY>`, read only by that target |
| Absent | An absent key means `off`; an absent ConfigMap means every target is off |
| Label | `reconcile.fluxcd.io/watch: Enabled`, so a change applies immediately, not at the next 1-minute interval |

A process on the cluster host writes this ConfigMap from the host's own configuration; this repo
never knows where the values come from. It derives each key from a parameter name:

**Key rule.** Take the parameter name, strip everything up to and including `targets/`, replace every `/`, `-` and `.` with `_`, uppercase the result, and prefix `TARGET_`: `targets/online-boutique` becomes `TARGET_ONLINE_BOUTIQUE`, and `targets/online-boutique/loadgen_replicas` becomes `TARGET_ONLINE_BOUTIQUE_LOADGEN_REPLICAS`.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ozlabs-targets
  namespace: flux-system
  labels:
    reconcile.fluxcd.io/watch: Enabled
data:
  TARGET_ONLINE_BOUTIQUE: "on"                 # quote it: unquoted on/off are YAML booleans
  TARGET_ONLINE_BOUTIQUE_LOADGEN_REPLICAS: "1"
```

Current keys:

| Key | Values | Default |
|---|---|---|
| `TARGET_ONLINE_BOUTIQUE` | `on` / `off` | `off` |
| `TARGET_ONLINE_BOUTIQUE_LOADGEN_REPLICAS` | integer ≥ 0 | `1` |

Use only `on` or `off` for a switch: any other value points the target at a path that does not exist,
and that target's Kustomization fails, without pruning anything, until the value is fixed.

**How a switch works.** `targets` builds `./targets` and substitutes the ConfigMap's values into each
`switch.yaml`, whose `path` becomes `./targets/<name>/on` or `./targets/<name>/off`. The switch
Kustomization substitutes again inside `on/` (sub-keys such as the load-generator replicas). Switching to
`off` renders nothing, so `prune` removes everything the target created, including its namespace.

**Reconcile on change.** kustomize-controller reconciles a Kustomization when a ConfigMap it
substitutes from changes, if that ConfigMap matches the controller's
`--watch-configs-label-selector`. In v2.9.6 the flag defaults to
`reconcile.fluxcd.io/watch=Enabled`; `flux-system/kustomization.yaml` sets it explicitly to that value
so the behaviour is pinned here. Events fire even though `substituteFrom` is marked `optional`,
including when the ConfigMap is deleted.

## Install (cloud-init)

Flux is installed from this repo's files at a **pinned commit**, with no bootstrap step and no
credentials on the node. Replace `<FLEET_COMMIT_SHA>` with the full SHA of the commit to install from:

```sh
kubectl apply -f https://raw.githubusercontent.com/yasinsd/ozlabs-fleet/<FLEET_COMMIT_SHA>/clusters/ozlabs-k3s/flux-system/gotk-components.yaml
kubectl wait --for=condition=Established --timeout=2m \
  crd/gitrepositories.source.toolkit.fluxcd.io crd/kustomizations.kustomize.toolkit.fluxcd.io
kubectl apply -f https://raw.githubusercontent.com/yasinsd/ozlabs-fleet/<FLEET_COMMIT_SHA>/clusters/ozlabs-k3s/flux-system/gotk-sync.yaml
```

The first file creates the `flux-system` namespace, the CRDs and the two controllers. The `wait` lets
the CRDs register before the second file creates the `flux-system` GitRepository and Kustomization.
From then on, Flux reconciles `./clusters/ozlabs-k3s` from `main`, including its own manifests: the
kustomize-controller patch is applied by that first reconcile. The cluster then runs `targets`, with
every target off until the ConfigMap switches one on.

## Add a target

1. Copy `targets/online-boutique/` to `targets/<name>/`, keeping its shape: `switch.yaml`,
   `off/kustomization.yaml` (`resources: []`) and `on/`.
2. In `switch.yaml`, set `metadata.name: <name>` and
   `path: ./targets/<name>/${TARGET_<NAME>:=off}`.
3. Put the target's objects under `on/`. Read sub-keys there as `${TARGET_<NAME>_<KEY>:=<default>}`;
   the switch Kustomization substitutes them. Do not add `postBuild` to a Kustomization that applies
   third-party manifests (like `online-boutique-app`), so upstream files are applied as published.
4. Add `- <name>/switch.yaml` to `targets/kustomization.yaml`.
5. Expose nothing outside the cluster: no `LoadBalancer` or `NodePort` Services.

## Change a pin

- **Flux:** with the flux CLI at the new version, run
  `flux install --export --components=source-controller,kustomize-controller --version=<version>`,
  replace `gotk-components.yaml` with the output unchanged, and update `VERSION` and the table above.
  Re-check the flag and label under *Reconcile on change* for that version. Running clusters upgrade
  from `main`; new clusters install from whatever commit cloud-init pins.
- **An application:** change `ref.commit` (full SHA) in the target's `on/source.yaml`.
- **The install commit:** the `<FLEET_COMMIT_SHA>` in cloud-init. It only decides which Flux version
  a new cluster starts from; the cluster follows `main` afterwards.

## Validate

```sh
kubectl kustomize clusters/ozlabs-k3s            # and flux-system, targets, targets/*/on, targets/*/off

# Render the app the way kustomize-controller does (components + patches). Substitute the
# sub-keys first, as the switch Kustomization does in the cluster:
sed 's/\${TARGET_ONLINE_BOUTIQUE_LOADGEN_REPLICAS:=1}/1/' \
  targets/online-boutique/on/release.yaml > /tmp/release.yaml
flux build kustomization online-boutique-app \
  --path <checkout of the app at its pinned commit>/kustomize \
  --kustomization-file /tmp/release.yaml --dry-run
```

Static builds keep the `${...}` placeholders; Flux substitutes them in the cluster. For schema checks,
run kubeconform with the Flux CRD schemas published with each Flux release (`crd-schemas.tar.gz`); the
`CustomResourceDefinition` objects in `gotk-components.yaml` have no schema in kubeconform's default
set and are skipped.

## Test a local checkout

A GitRepository URL must be `http`, `https` or `ssh`; a local filesystem path is not accepted. To test
unpushed changes, put them on a throwaway branch whose `gotk-sync.yaml` points at where the test cluster
can fetch them: either a temporary branch on this repo (`ref.branch`), or the checkout served over
Git's smart HTTP protocol (`git http-backend`) at an address the cluster can reach. The change must be
committed on that branch; otherwise the first `flux-system` reconcile re-applies the committed
`gotk-sync.yaml` and points Flux back at `main`.

## Licence

MIT, see `LICENSE`.
