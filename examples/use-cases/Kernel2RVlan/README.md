# Test kernel to remote vlan connection

This example shows that NSC can establish a remote vlan connection.
NSCs are using the `kernel` mechanism to connect to its local forwarder.
Forwarders are using the `vlan` remote mechanism.

## Requires

Make sure that you have completed steps from [remotevlan](../../remotevlan) setup.

## Run

Create test namespace:

```bash
NAMESPACE=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/7113942326f9001fa67b7a9effdf38d4eba2dbdd/examples/use-cases/namespace.yaml)[0])
NAMESPACE=${NAMESPACE:10}
```

Create customization file:

```bash
cat > kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ${NAMESPACE}

bases:
- ../../../apps/nsc-kernel


patchesStrategicMerge:
- patch-nsc.yaml
EOF
```

Create NSC patch:

```bash
cat > patch-nsc.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nsc-kernel
spec:
  replicas: 2
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nsc-kernel
            topologyKey: "kubernetes.io/hostname"
      containers:
        - name: nsc
          env:
            - name: NSM_NETWORK_SERVICES
              value: kernel://finance-bridge/nsm-1
EOF
```

Deploy NSC:

```bash
kubectl apply -k .
```

Wait for applications ready:

```bash
kubectl -n ${NAMESPACE} wait --for=condition=ready --timeout=1m pod -l app=nsc-kernel
```

Get NSC pod:

```bash
NSCS=($(kubectl get pods -l app=nsc-kernel -n ${NAMESPACE} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'))
```

Ping from NSC address from each NSC:

```bash
kubectl exec ${NSCS[0]} -n ${NAMESPACE} -- ping -c 4 172.10.0.1
kubectl exec ${NSCS[1]} -n ${NAMESPACE} -- ping -c 4 172.10.0.1
kubectl exec ${NSCS[0]} -n ${NAMESPACE} -- ping -c 4 172.10.0.2
kubectl exec ${NSCS[1]} -n ${NAMESPACE} -- ping -c 4 172.10.0.2
```

## Cleanup

Delete ns:

```bash
kubectl delete ns ${NAMESPACE}
```
