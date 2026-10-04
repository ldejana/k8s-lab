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


## Readiness and liveness

### Broken readiness

Readiness OK -> complete rollout and redirect the traffic

Readiness probe NOT OK -> latest replica set stalls, the traffic remains on the previous RS

No rollback, the new version just never takes over.

### Broken liveness 

constant restart of pods, no rollback.
it has exponential backoff, but kubernetes restarts indefinitely.

## Selector break

service.yml -> spec.selector points to random value

result: pods not affected, but cannot reach the pods anymore

```
kubectl apply -f 01-k8s/service.yml
kubectl describe svc podinfo -> Endpoints = None
kubectl port-forward svc/podinfo 9898:9898 -> times out, no matching pods
curl localhost:9898 -> fail
```

## Imposter pod

pull images and load to cluster

```
docker pull nginx:1.27 && kind load docker-image nginx:1.27 --name lab
docker pull curlimages/curl:8.10.1 && kind load docker-image curlimages/curl:8.10.1 --name lab
```

Create imposter pod (imposter is just pod name)
`kubectl run imposter --image=nginx:1.27 --labels=app=podinfo`

Result:

1. Service redirects the traffic to the imposter pod as the label matches
2. RS does not delete any pod to bring it back to the number of replicas selected.
   If pod was to match the same hash of the RS, replica set would adopt it and adjust the number of pods (likely delete it as it's the newest). The match is not done based on the pod name, but on the labels.
