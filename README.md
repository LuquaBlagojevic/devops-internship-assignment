# DevOps Internship Assignment

A Helm chart that deploys Nginx to Kubernetes using a Deployment and a Service.

## Prerequisites

- Docker Desktop with Kubernetes enabled
- kubectl
- Helm

The commands below use the `docker-desktop` Kubernetes context.

## Validate the chart

```bash
helm lint charts/webapp
helm template internship charts/webapp
```

`helm lint` checks the chart for potential problems.
`helm template` renders the Kubernetes manifests without installing them.

## Install

Run from the repository root:

```bash
helm install internship charts/webapp \
  --kube-context docker-desktop \
  --namespace internship \
  --create-namespace \
  --wait \
  --timeout 5m
```

This creates a Helm release named `internship` in the `internship` namespace.

## Configuration

The configurable settings are defined in `charts/webapp/values.yaml`.

| Value | Default | Purpose |
|---|---|---|
| `applicationName` | `webapp` | Deployment, Service, and container name |
| `image.repository` | `nginx` | Container image repository |
| `image.tag` | `"1.28.0"` | Container image tag |
| `replicaCount` | `2` | Desired number of Pods |
| `applicationPort` | `80` | Port on which the application listens |
| `service.type` | `ClusterIP` | Service type: `ClusterIP` or `NodePort` |
| `service.port` | `80` | Port exposed by the Service |

The application must actually listen on `applicationPort`.
Changing this value does not reconfigure Nginx itself.

After editing `values.yaml`, apply the changes:

```bash
helm upgrade internship charts/webapp \
  --kube-context docker-desktop \
  --namespace internship \
  --wait \
  --timeout 5m
```

Values can also be overridden for an individual install or upgrade
using `--set`, for example `--set replicaCount=2`.

To preview the NodePort configuration without deploying it:

```bash
helm template internship charts/webapp --set service.type=NodePort
```

With `NodePort`, Kubernetes automatically allocates the node port.

## Access the application

Forward a local port to the Service:

```bash
kubectl --context docker-desktop -n internship \
  port-forward service/webapp 8080:80
```

Keep this terminal open and visit http://localhost:8080.
The default Nginx welcome page should appear.

Alternatively, run in a second terminal:

```bash
curl -I http://localhost:8080
```

Expected response: `HTTP/1.1 200 OK`.

Press Ctrl+C to stop port-forwarding. The application continues
running in Kubernetes.

## Verify the deployment

```bash
kubectl --context docker-desktop -n internship \
  get deployments,pods,services
```

With the default configuration, expect:

- Deployment `webapp` with `READY 2/2`.
- Two Pods with status `Running` and `READY 1/1`.
- Service `webapp` with type `ClusterIP` and port `80/TCP`.

Check the rollout and Helm release:

```bash
kubectl --context docker-desktop -n internship \
  rollout status deployment/webapp --timeout=120s

helm list --kube-context docker-desktop --namespace internship
```

During local verification, the application was accessed through the
Service using port-forwarding. Scaling from one to two replicas using
`helm upgrade` was also verified successfully.

## Troubleshooting a Pod that fails to start

First, find the affected Pod:

```bash
kubectl --context docker-desktop -n internship get pods
```

Replace `<pod-name>` below with its actual name.

Inspect its state and events:

```bash
kubectl --context docker-desktop -n internship describe pod <pod-name>
```

Check application logs:

```bash
kubectl --context docker-desktop -n internship logs <pod-name>
```

If the container has restarted, inspect the previous instance:

```bash
kubectl --context docker-desktop -n internship \
  logs <pod-name> --previous
```

Typical cases:

- `ErrImagePull` or `ImagePullBackOff`: inspect events and check the
  image repository, tag, registry access, and network connectivity.
  Application logs may not exist because the container has not started.
- `CrashLoopBackOff`: inspect current and previous logs for application
  errors, then check configuration and the termination reason in
  `describe pod`.
- `Pending`: inspect scheduling events for insufficient resources,
  node availability, or other scheduling constraints.

After correcting the configuration, run `helm upgrade` again and
verify the rollout.

## Uninstall

```bash
helm uninstall internship \
  --kube-context docker-desktop \
  --namespace internship
```

This removes the release's Deployment and Service. The namespace remains.