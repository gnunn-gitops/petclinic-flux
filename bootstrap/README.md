# bootstrap

Cluster entrypoint: the `GitRepository` sources and the Flux `Kustomization`
objects that apply everything else, plus the GitHub webhook receiver.

**This directory is not reconciled by Flux.** Nothing has `./bootstrap/base` as
its path, so changes here only take effect when you apply them by hand:

```sh
oc apply -k bootstrap/base
```

Everything under `apps/` *is* Flux-managed and needs only a `git push`.

## GitHub webhook

`flux-webhook-receiver.yaml` declares a `Receiver` and an edge-terminated
`Route`. Pushes to the repository then reconcile the matching `GitRepository`
immediately instead of waiting up to the 1m polling interval. Polling remains
the fallback for missed deliveries.

Two things cannot live in git and must be done once per cluster:

### 1. The HMAC token Secret

Shared between the `Receiver` and the GitHub hook, so GitHub can prove a
delivery is genuine.

```sh
oc create secret generic flux-webhook-token -n flux-system \
  --from-literal=token="$(python3 -c 'import secrets; print(secrets.token_hex(32))')"
```

It is a plain Secret rather than an `ExternalSecret` because the `petclinic`
`ClusterSecretStore` only serves namespaces labelled `tenant: petclinic`, and
`flux-system` is not one of them. Labelling `flux-system` would hand the whole
Flux namespace access to the Doppler project, which is a bigger grant than this
one token is worth. If you would rather manage it through Doppler, add the key
there and label the namespace.

### 2. Register the hook in GitHub

The path is a hash of the Receiver, so read it back rather than guessing:

```sh
HOST=$(oc get route flux-webhook -n flux-system -o jsonpath='{.spec.host}')
WPATH=$(oc get receiver petclinic-github -n flux-system -o jsonpath='{.status.webhookPath}')
TOKEN=$(oc get secret flux-webhook-token -n flux-system -o jsonpath='{.data.token}' | base64 -d)

python3 -c "
import json,sys
print(json.dumps({'name':'web','active':True,'events':['push'],
 'config':{'url':sys.argv[1],'content_type':'json','secret':sys.argv[2],'insecure_ssl':'0'}}))
" "https://\$HOST\$WPATH" "\$TOKEN" > /tmp/hook.json

gh api -X POST repos/gnunn-gitops/petclinic-flux/hooks --input /tmp/hook.json
rm -f /tmp/hook.json
```

`insecure_ssl: 0` is deliberate. The Route uses the ingress wildcard
certificate, which on this cluster is issued by a public CA, so GitHub can
verify it. On a cluster with a self-signed ingress certificate this hook would
fail until you either install a trusted certificate or set `insecure_ssl: 1`.

**The Route host and the webhook path both change when the cluster is
rebuilt.** Delete the stale hook and re-run the above.

### Verifying

```sh
# GitHub's view — ping should be 200
gh api repos/gnunn-gitops/petclinic-flux/hooks --jq '.[] | "\(.id) \(.config.url)"'
gh api repos/gnunn-gitops/petclinic-flux/hooks/<id>/deliveries \
  --jq '.[] | "\(.event) \(.status_code) \(.delivered_at)"' | head

# Flux's view — this timestamp jumps on every delivery
oc get gitrepository petclinic -n flux-system \
  -o jsonpath='{.status.lastHandledReconcileAt}{"\n"}'

oc get receiver petclinic-github -n flux-system
```

A delivery returning 404 usually means the Receiver was recreated and the path
changed; 401 means the token in GitHub and the one in the Secret disagree.
