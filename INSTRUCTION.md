# Deploying the ToDo app to Kubernetes

These instructions assume you have `kubectl` configured for a Kubernetes cluster and that the cluster has Metrics Server installed. The HPA needs Metrics Server to read CPU and memory usage.

## Deploy

From the repository root, create the namespace and apply the Deployment and autoscaler:

```sh
kubectl create namespace mateapp
kubectl apply -f deployment.yml
kubectl apply -f hpa.yml
```

The Deployment starts two replicas. Wait until both are ready:

```sh
kubectl rollout status deployment/todoapp -n mateapp
kubectl get pods -n mateapp
```

Create a NodePort Service to expose the Deployment outside the cluster:

```sh
kubectl expose deployment todoapp \
  --name=todoapp \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  -n mateapp
```

Find the assigned NodePort and a reachable node address:

```sh
kubectl get service todoapp -n mateapp
kubectl get nodes -o wide
```

Open `http://<node-address>:<node-port>/` in a browser, substituting the node's address and the port shown in the `todoapp` Service's `PORT(S)` column. On a local Minikube cluster, use `minikube service todoapp -n mateapp --url` to get a reachable URL.

## Resource requests and limits

Each pod requests `100m` CPU and `128Mi` memory. These are modest starting reservations for this lightweight Django app, so Kubernetes can schedule two baseline replicas without reserving excessive node capacity. The requests also provide the baseline used by the HPA's utilization targets.

The per-container limits are `500m` CPU and `256Mi` memory. They allow several times the baseline CPU and memory during bursts while bounding each container's resource consumption. Observe actual usage in the target cluster and adjust these starting values if normal workload approaches the limits.

## Horizontal Pod Autoscaler

The HPA targets the Deployment and keeps between two and five replicas. The minimum of two maintains the required idle capacity and gives the app a second pod for availability; the maximum of five puts a bound on scale-out.

Scaling is based on both CPU and memory: the targets are 70% of requested CPU and 80% of requested memory, averaged across the pods. These targets leave some headroom before the container's limits, while allowing the HPA to react to either compute-heavy or memory-heavy load. When both metrics are available, Kubernetes calculates a desired replica count for each and scales to satisfy the metric calling for more replicas. Check that both metrics are reported with `kubectl top pods -n mateapp` and inspect autoscaler decisions with `kubectl describe hpa todoapp -n mateapp`.

## Rolling update strategy

`maxSurge: 1` allows one temporary extra pod during an update, so the Deployment can bring up a replacement before removing an old pod. `maxUnavailable: 0` prevents the rollout controller from intentionally reducing the available replica count below the desired count. With two replicas, this favors availability at the cost of temporarily running a third pod; the readiness probe ensures a replacement is ready before it receives traffic and the rollout proceeds.
