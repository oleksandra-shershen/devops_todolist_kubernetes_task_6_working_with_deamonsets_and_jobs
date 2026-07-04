# Deploying the ToDo App to Kubernetes

Manifests in this folder, all in the `mateapp` namespace:
- `namespace.yml` — the namespace itself
- `deployment.yml` — the app
- `hpa.yml` — autoscaling
- `service.yml` — exposes the app (NodePort)

## Deploy

```bash
kubectl apply -f namespace.yml
kubectl apply -f .
```

Check it came up:

```bash
kubectl -n mateapp get pods
kubectl -n mateapp get hpa
```

You should see 2 `todoapp` pods running.

## Resources: 100m/128Mi requests, 250m/256Mi limits

It's a small app with no heavy background work, so it doesn't need much
at idle — the request reflects that. The limit is set well above the
request (~2.5x) just to give each pod room for short spikes without
getting OOMKilled or throttled too aggressively. Nothing scientific
here — a sane starting point, to be adjusted later once you can see
real usage (`kubectl top pods`).

## HPA: min 2 / max 5, scale on CPU and memory at 70%

Min 2 keeps the "always 2 pods at idle" requirement and gives some
redundancy for free. Max 5 gives enough headroom for traffic bursts
without letting things scale out of control. Both CPU and memory are
tracked because either one could be the bottleneck for this app, and
70% is a middle-ground threshold — reacts early enough, but doesn't
trigger on every small fluctuation.

## Rolling update: maxSurge 1, maxUnavailable 0

`maxUnavailable: 0` means we never go below the current pod count
during a deploy, so together with `minReplicas: 2` there are always at
least 2 pods serving traffic. `maxSurge: 1` adds just one extra pod at
a time to move the rollout forward — enough to make progress, cheap
enough not to double resource usage mid-deploy.

## Accessing the app

`service.yml` exposes it as NodePort `30080`:

```bash
kubectl -n mateapp get nodes -o wide
```

then open `http://<node-ip>:30080/`. On minikube: `minikube service todoapp -n mateapp`.

Or skip the Service entirely and port-forward straight to the deployment:

```bash
kubectl -n mateapp port-forward deployment/todoapp 8080:8080
```

then open `http://localhost:8080/`.

## DaemonSet and CronJob

Two more manifests, also in `mateapp`, both assume the `todoapp` Service
(from `service.yml`) is already deployed — they hit it internally at
`http://todoapp.mateapp.svc.cluster.local`.

- `daemonset.yml` — one `busyboxplus:curl` pod per node, curling the
  app every 5 seconds in a loop.
- `cronjob.yml` — runs every 4 minutes, hits `/api/health`, keeps 10
  successful and 5 failed job records, and allows overlapping runs
  (`concurrencyPolicy: Allow`).

### Deploy

```bash
kubectl apply -f daemonset.yml
kubectl apply -f cronjob.yml
```

### Validate

Check the DaemonSet has a pod per node and is healthy:

```bash
kubectl -n mateapp get daemonset todoapp-curl
kubectl -n mateapp get pods -l app=todoapp-curl
```

Look at its logs — you should see a response (or at least a completed
request) roughly every 5 seconds:

```bash
kubectl -n mateapp logs -l app=todoapp-curl -f
```

Check the CronJob is scheduling and see its run history:

```bash
kubectl -n mateapp get cronjob todoapp-health-check
kubectl -n mateapp get jobs
kubectl -n mateapp get pods
```

After a few runs you should see up to 10 completed pods and up to 5
failed ones kept around (older ones get cleaned up automatically).
Check a specific run's logs:

```bash
kubectl -n mateapp logs job/<job-name>
```

A successful run's log should just be the JSON/text response from
`/api/health`.