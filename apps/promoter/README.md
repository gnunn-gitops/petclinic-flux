# petclinic promoter configuration

`base/` is applied by the `petclinic-promoter` Flux Kustomization
(`bootstrap/base/petclinic-promoter-kustomization.yaml`) into the
`petclinic-promoter` namespace.

## Promotion gates

| Resource | Key | Reports on | What it gates |
| --- | --- | --- | --- |
| `github-checks-commit-status.yaml` | `github-required-checks` | proposed | GitHub's actually-required checks on the promotion PR |
| `flux-health-commit-status.yaml` | `flux-health` | active | Flux has applied the promoted SHA and the Argo Rollout is healthy |
| `dependency-commit-status.yaml` | `dependents-successful` | proposed | Ordering — dev, then stage, then prod |

`flux-health` is wired into `promotion-strategy.yaml` as a top-level
`activeCommitStatuses` entry, so it applies to all three environments.
`DependentsSuccessfulCommitStatus` (the strategy's `orderCommitStatusRef`)
requires each environment to be successful for what is live on its active
branch before the next one promotes — which is what makes the Flux health
result block the next environment.

It pairs with the `healthChecks` / `healthCheckExprs` added to the three
environment Kustomizations in `bootstrap/base/`: those make Flux's `Ready`
condition mean "the petclinic Argo Rollout reached phase `Healthy`", so this
gate inherits that meaning for free.

## Prerequisite: let the promoter trust the cluster CA

`flux-health` reads the Flux `Kustomization` objects from
`https://kubernetes.default.svc`, whose serving certificate is signed by a
cluster-internal CA that the promoter's HTTP client does not trust by default.
Apply the one-time patch:

```sh
oc patch deployment promoter-controller-manager -n promoter-system \
  --patch-file apps/promoter/controller-ca-trust-patch.yaml
```

See the comments in `controller-ca-trust-patch.yaml` for why `SSL_CERT_DIR` is
used rather than `SSL_CERT_FILE`, and why the problem cannot be solved from
within the `WebRequestCommitStatus` itself.

The patch is **not** managed by Flux: the promoter controller is installed with
`kubectl apply -k` from the upstream repo, so re-apply the patch after a
promoter upgrade or reinstall.

### Verifying

```sh
# the gate's own view, per environment
oc get webrequestcommitstatus flux-health -n petclinic-promoter \
  -o jsonpath='{range .status.environments[*]}{.branch}{"  "}{.phase}{"  "}{.successOutput.summary}{"\n"}{end}'

# what it is comparing against
oc get kustomization -n flux-system -o custom-columns=\
NAME:.metadata.name,REVISION:.status.lastAppliedRevision,READY:.status.conditions[0].status
```

A `403` in `summary` means the patch is applied but the token is not authorized;
a persistent `pending` with no `statusCode` usually means the HTTP call itself
is failing — check the controller logs for an x509 error, which means the CA
patch has not taken effect.
