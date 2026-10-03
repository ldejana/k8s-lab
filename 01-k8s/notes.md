# Notes

**Rule:** before anything, `kubectl config current-context` must say `kind-lab`.

## Deployment

- terminal 1: `kubectl get pods -w` (if output looks stale, the watch may have dropped; run `kubectl get pods` fresh)
- terminal 2: `kubectl apply -f 01-k8s/deployment.yaml`, then observe the pod appear
- `kubectl delete pod <pod-name>`, then observe the pod disappear **and a replacement appear with a new name**

**Lesson:** pods are disposable. Replacements get a new name and a new IP. Never depend on a pod's name or IP; use labels and Services.

### Pod names

```
podinfo-85756bd858-v6fz6
podinfo-85756bd858-2vs9j
<deployment-name>-<pod-template-hash>-<random-per-pod>
```

- `pod-template-hash`: hash of the pod template (`spec.template` in `deployment.yaml`); it identifies the ReplicaSet
- Change anything under `spec.template` → new hash → new ReplicaSet → pods replaced one by one
- The old ReplicaSet is kept, scaled to 0; `kubectl rollout undo` uses it

### Scaling to 3 replicas

Changed `replicas: 3` in the YAML, `kubectl apply` printed `configured` (= API server accepted the new desired state).
Seemed like nothing happened because my watch terminal was stale.

Check the chain **Deployment → ReplicaSet → Pods**, in order:
- `kubectl get deployment podinfo` (READY shows actual/desired, e.g. `1/3`)
- `kubectl get replicaset`
- `kubectl get pods`
- If pods are missing: `kubectl get events --sort-by=.lastTimestamp`

## Problem: `ImagePullBackOff`

- `ImagePullBackOff` = the pull failed and Kubernetes waits before retrying. The real cause is in the **Events** section of `kubectl describe pod <pod-name>`.
- My cause: `dial tcp 140.82.114.33:443: i/o timeout`. DNS worked (got an IP), but the kind node can't connect to ghcr.io (network problem, not a wrong tag).
- Workaround: `docker pull <image>` on the host, then `kind load docker-image <image> --name lab` (copies the image from host Docker into the node's containerd).
- In real clusters: images always come from a registry (private registry, pull-through mirror). Nobody loads images onto nodes by hand.


### `imagePullPolicy` defaults

- Specific tag (e.g. `:6.7.0`): `IfNotPresent` (use the local copy if it exists)
- `:latest` **or no tag at all**: `Always` (pull every time)
- That's one reason `:latest` is a bad idea in deployments.

## Service

- `kubectl apply -f 01-k8s/service.yaml`
- `kubectl describe svc podinfo`
- `Endpoints: 10.244.0.6:9898,10.244.0.7:9898,10.244.0.8:9898` = 3 pods matching the selector `app=podinfo`
- The Service IP (`10.96.247.8`, ClusterIP) stays fixed while pod IPs change. That stable address is why Services exist.


## Rollout and rollback

Changing the template creates a new replica set.

ex: added readiness probe created a new replica set and redirected the traffic.

broken readiness probe:
- a new replica set created but probe does not pass so the latest rs stalls
- the previous replica set is then still in use

revert the readiness probe:
- results in the same template (rs hash) -> same replica set that is already running

A broken readiness probe stalls the rollout; old pods keep serving. Nothing is rolled back, the new version just never takes over. Reverting the template to a previous version reuses that version's ReplicaSet.