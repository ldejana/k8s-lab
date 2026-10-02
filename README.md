# k8s-lab

# k8s-lab

Hands-on lab for learning Kubernetes, Helm and Argo CD from first principles.

## The one idea that ties all three together

All three tools work on the same pattern: **declare the desired state, and let a controller keep reality matching it.**

| Layer | Desired state lives in | What reconciles it |
|---|---|---|
| Kubernetes | YAML objects in the API server | Controllers (e.g. Deployment controller) |
| Helm | `values.yaml` + templates, rendered into YAML | Nothing continuous: Helm renders and applies once per install/upgrade |
| Argo CD | Git | Argo CD, continuously |

When something breaks, first ask: **which layer is it in?**

## Repo layout

```
01-k8s/      hand-written manifests (no Helm)
02-helm/     my own chart
03-argocd/   Argo CD Application manifest
notes.md     symptom -> command -> cause log, plus open questions
```

Test app: [podinfo](https://github.com/stefanprodan/podinfo) (`ghcr.io/stefanprodan/podinfo`).
It listens on port 9898 and has `/healthz` and `/readyz` endpoints, so it's good for practicing probes.

---

## Session 0: Setup (15 min, tonight)

- [x] Docker running (Docker Desktop, OrbStack, or Colima)
- [x] Install tools: `brew install kind kubectl helm` (or see each tool's install docs)
- [x] `kind create cluster --name lab`, then `kubectl get nodes` shows `Ready`
- [x] Create a **public** GitHub repo `k8s-lab` (public = Argo CD can read it without credentials in session 3)
- [x] Add this README, commit, push
- [x] Stop. Don't start session 1.

## Session 1: Kubernetes

**First-principles question:** What happens between `kubectl apply` and a running pod?

- [x] `kubectl get all -A`: what's already running in an empty cluster? Why?
- [x] Hand-write `01-k8s/deployment.yaml` for podinfo (1 replica) and apply it
- [x] `kubectl get pods -w` in one terminal, then delete the pod in another and watch it come back (this is reconciliation)
- [x] Change replicas to 3 **in the YAML** and re-apply (declarative, not `kubectl scale`)
- [x] Write `01-k8s/service.yaml`, `kubectl port-forward svc/podinfo 9898:9898`, `curl localhost:9898`
- [ ]  ▶ Warm-up (5 min): docker start lab-control-plane, kubectl get pods. Are the 3 pods back? kubectl port-forward svc/podinfo 9898:9898, then curl localhost:9898
- [ ] Add readiness and liveness probes plus resource requests/limits
- [ ] **Break it on purpose** and log each one in `notes.md` (symptom, command used, cause):
  - [x] Wrong image tag (expect `ImagePullBackOff`)
  - [ ] Wrong liveness probe path (expect restarts)
  - [ ] Wrong readiness probe path (expect pod not `Ready`, no traffic)
  - [ ] Service selector that doesn't match pod labels (expect `kubectl get endpoints` to be empty)
- [ ] Commit
- - [ ] **Imposter pod: labels are a contract** (about 15 min). Start a bare nginx pod with the label `app=podinfo` and see who notices.
  - [ ] Load the images first (the node's network can't pull them):
    ```bash
    docker pull nginx:1.27 && kind load docker-image nginx:1.27 --name lab
    docker pull curlimages/curl:8.10.1 && kind load docker-image curlimages/curl:8.10.1 --name lab
    ```
  - [ ] **Predict before running anything:** Does the Service send it traffic? Does the ReplicaSet delete a pod to get back to 3? What fraction of requests fail?
  - [ ] Start it: `kubectl run imposter --image=nginx:1.27 --labels=app=podinfo`
  - [ ] `kubectl get pods --show-labels`: compare the imposter's labels with the podinfo pods' labels
  - [ ] `kubectl describe svc podinfo`: how many Endpoints now?
  - [ ] `kubectl get replicaset -o wide`: read the SELECTOR column. Why is the imposter ignored?
  - [ ] Send traffic from inside the cluster (port-forward sticks to one pod, so it won't show this):
    ```bash
    kubectl run tester --rm -it --image=curlimages/curl:8.10.1 --restart=Never -- \
      sh -c 'for i in $(seq 1 20); do curl -s -m 2 -o /dev/null podinfo:9898 && echo ok || echo FAIL; done'
    ```
    How many `FAIL`s, and why? (Hint: which port does nginx listen on?)
  - [ ] Clean up with `kubectl delete pod imposter` and confirm the Endpoints are back to 3
  - [ ] Log in `notes.md`: what the Service checks (labels only), what the ReplicaSet checks (labels including `pod-template-hash`), and why the imposter was treated as ready immediately
- [ ] `kubectl rollout undo deployment/podinfo`: what did it roll back to?
- [ ] Write my answer to the first-principles question in `notes.md`
- [ ] Commit

**Done when:** I can answer the first-principles question out loud without notes.


**Done when:** I can answer the first-principles question out loud without notes.

## Session 2: Helm

**First-principles question:** What does Helm add on top of plain YAML, and what does it *not* do?

- [ ] `helm create` a scratch chart and read every generated file. Compare it with my hand-written YAML.
- [ ] Build my own minimal chart in `02-helm/podinfo` from my session-1 manifests, parametrizing image tag, replicas and resources
- [ ] `helm template` before and after a values change. Predict the diff first, then check.
- [ ] Add a `values-prod.yaml` override (environment pattern)
- [ ] `helm install`, `helm upgrade`, `helm history`, `helm rollback`
- [ ] Break a template (bad indentation or a missing value) and read the error
- [ ] Commit

**Done when:** I can predict rendered output from a values change.

## Session 3: Argo CD

**First-principles question:** What does Argo CD do that `helm upgrade` from a CI pipeline doesn't?

- [ ] Install Argo CD (namespace `argocd`, stable `install.yaml`), port-forward the UI, log in
- [ ] Write `03-argocd/application.yaml` pointing at `02-helm/podinfo` in this GitHub repo, then apply it
- [ ] Push a values change to Git and watch it sync
- [ ] Create drift: `kubectl edit` the Deployment by hand and watch it go `OutOfSync`
- [ ] Enable `selfHeal` and repeat. What changed?
- [ ] Push a broken template and find the sync error in the UI
- [ ] Commit

**Done when:** I can explain GitOps and its trade-offs (e.g. hotfixes must go through Git).

## Session 4: Real service at work

- [ ] Pick one service my team owns and trace it: Argo `Application`, chart, per-environment values, then the live objects in the cluster
- [ ] Note what differs from my lab (secrets handling, promotion flow, conventions)
- [ ] Turn open questions in `notes.md` into the agenda for a 30-min session with the platform team
