<h1 align="center"><img src="https://github.com/user-attachments/assets/f28e04e0-0610-412c-9c58-fa53706a9c91"> Leader Election</h1>
<p align="center">
  <img src="https://img.shields.io/github/languages/code-size/rkrmr33/leader-election?style=flat-square">
  <a href="https://github.com/rkrmr33/leader-election/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/rkrmr33/leader-election?style=flat-square">
  </a>
  <img src="https://img.shields.io/github/check-runs/rkrmr33/leader-election/main">
  <a href="https://github.com/rkrmr33/leader-election/releases/latest">
    <img src="https://img.shields.io/github/v/release/rkrmr33/leader-election">
  </a>
</p>

## Overview

The **Leader Election** project is a Kubernetes-native solution that acts as a
sidecar container,
leveraging the Kubernetes Lease object. This enables seamless leader election
for distributed systems,
allowing coordination without needing custom leader election implementation in
application code.

## Running the Example Manifests

### Prerequisites

Before running the example manifests, ensure you have the following tools installed:

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)

### Running the Manifests

To run the example manifests provided in the `manifests` directory:

#### With Kustomization

Apply the manifests using `kubectl` with the `-k` flag:

```bash
kubectl apply -k manifests/
```

#### Without Kustomization

Apply the manifests in the correct order to ensure that all resources are
created successfully:

   ```bash
   kubectl apply -f manifests/deployment.yaml
   kubectl apply -f manifests/role.yaml
   kubectl apply -f manifests/role-binding.yaml
   kubectl apply -f manifests/service-account.yaml
   ```

### Testing Leader Election

The deployment creates a debug pod named `net-debug`, pre-installed with the
   `curl` command-line tool. The pod runs a command to sleep for 100,000
seconds, effectively keeping it alive for manual testing and debugging purposes.

To test leader election behavior, check the current leader:

  ```bash
  POD_NAME=$(kubectl get pods -l app=leader-elector -o jsonpath='{.items[0].metadata.name}')
  kubectl exec \
    -it "$POD_NAME" \
    -c net-debug \
    -- /bin/bash -c "curl -s localhost:4040/api/leader | jq -r '.leader'"
  ```

Observe the current leader in a different pod (make sure to replace `<POD_NAME>`
with the actual pod name):

  ```bash
  kubectl exec \
    -it "<POD_NAME>" \
    -c net-debug \
    -- /bin/bash -c 'while sleep 1; do curl -s localhost:4040/api/leader; echo ""; done'
  ```

In a different session, kill the current leader pod. After about 30 seconds, you
should observe that a new leader has been elected.

## Configuration

### Environment Variables and Paths

The application allows for the following environment variables and paths to be configured:

| Environment Variable   | Description                                                                                     |
| ---------------------- | -----------------------------------------------------------------------------                 |
| `NAMESPACE`            | The namespace in the Kubernetes cluster. Typically fetched dynamically via a field reference. |
| `POD_NAME`             | The name of the pod running the container. Also dynamically fetched in most cases.            |
| `LEASE_NAME`           | The name of the lease used for the leader election process.                                   |
| `LEASE_DURATION`       | Duration for which a leader holds the lease. Example: `10s`.                                  |
| `LEASE_RENEW_DURATION` | Time interval to attempt lease renewal by the leader. Example: `5s`.                          |

Ensure that these are correctly set according to your deployment requirements.

### Endpoints

- `/healthz`: Configured as the liveness probe endpoint.
- `/readyz`: Configured as the readiness probe endpoint.
- `/api/leader`: Exposes a REST API endpoint to get information about the current leader.

## Using as a Sidecar Container

## Deployment as a Sidecar Container

Add the following container to your deployment:

```yaml
- name: leader-elector
  image: quay.io/roikramer120/leader-elector:v0.0.1
  command:
    - leader-elector
  args:
    - --id=$(POD_NAME)
    - --lease-name=$(LEASE_NAME)
    - --namespace=$(NAMESPACE)
    - --lease-duration=$(LEASE_DURATION)
    - --lease-renew-duration=$(LEASE_RENEW_DURATION)
  env:
    - name: NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: LEASE_NAME
      value: example
    - name: LEASE_DURATION
      value: 10s
    - name: LEASE_RENEW_DURATION
      value: 5s
  securityContext:
    allowPrivilegeEscalation: false
  livenessProbe:
    httpGet:
      path: /healthz
      port: 4040
    initialDelaySeconds: 15
    periodSeconds: 20
  readinessProbe:
    httpGet:
      path: /readyz
      port: 4040
    initialDelaySeconds: 5
    periodSeconds: 10
  resources:
    limits:
      cpu: 200m
      memory: 200Mi
    requests:
      cpu: 100m
      memory: 100Mi
  imagePullPolicy: IfNotPresent
```

> **Note:** Ensure that the pod service account is bound to a role with at least the following scopes:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/name: leader-elector
  name: leader-elector
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: get
      list
      watch
      create
      update
      patch
      delete
  - apiGroups:
      - ""
    resources:
      - events
    verbs: create
      patch
```

## Contributing

If you are interested in contributing to the Leader Election project, please follow these steps:

1. Fork the repository and create your branch from `main`.
2. Test your code thoroughly.
3. Submit a pull request with a detailed explanation of your changes.

We welcome all kinds of contributions: documentation, bug reports, feature requests, and code improvements.
