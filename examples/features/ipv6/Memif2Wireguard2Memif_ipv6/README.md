# Test memif to wireguard to memif connection

This example shows that NSC and NSE on the different nodes could find and work with each other using IPv6.

NSC and NSE are using the `memif` mechanism to connect to its local forwarder.
Forwarders are using the `wireguard` mechanism to connect with each other.

## Run

Create test namespace:
```bash
NAMESPACE=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/8b7093ac5b240e72edceac5224be3d7605bc8957/examples/features/namespace.yaml)[0])
NAMESPACE=${NAMESPACE:10}
```

Get nodes exclude control-plane:
```bash
NODES=($(kubectl get nodes -o go-template='{{range .items}}{{ if not .spec.taints  }}{{index .metadata.labels "kubernetes.io/hostname"}} {{end}}{{end}}'))
```

Create customization file:
```bash
cat > kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ${NAMESPACE}

bases:
- https://github.com/networkservicemesh/deployments-k8s/apps/nsc-memif?ref=8b7093ac5b240e72edceac5224be3d7605bc8957
- https://github.com/networkservicemesh/deployments-k8s/apps/nse-memif?ref=8b7093ac5b240e72edceac5224be3d7605bc8957

patchesStrategicMerge:
- patch-nsc.yaml
- patch-nse.yaml
EOF
```

Create NSC patch:
```bash
cat > patch-nsc.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nsc-memif
spec:
  template:
    spec:
      containers:
        - name: nsc
          env:
            - name: NSM_NETWORK_SERVICES
              value: memif://icmp-responder-ip/nsm-1

      nodeName: ${NODES[0]}
EOF
```
Create NSE patch:
```bash
cat > patch-nse.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nse-memif
spec:
  template:
    spec:
      containers:
        - name: nse
          env:
            - name: NSM_CIDR_PREFIX
              value: 2001:db8::/116
            - name: NSM_PAYLOAD
              value: IP
            - name: NSM_SERVICE_NAMES
              value: icmp-responder-ip
      nodeName: ${NODES[1]}
EOF
```

Deploy NSC and NSE:
```bash
kubectl apply -k .
```

Wait for applications ready:
```bash
kubectl wait --for=condition=ready --timeout=1m pod -l app=nsc-memif -n ${NAMESPACE}
```
```bash
kubectl wait --for=condition=ready --timeout=1m pod -l app=nse-memif -n ${NAMESPACE}
```

Find NSC and NSE pods by labels:
```bash
NSC=$(kubectl get pods -l app=nsc-memif -n ${NAMESPACE} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```
```bash
NSE=$(kubectl get pods -l app=nse-memif -n ${NAMESPACE} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

Ping from NSC to NSE:
```bash
result=$(kubectl exec "${NSC}" -n "${NAMESPACE}" -- vppctl ping 2001:db8:: repeat 4)
echo ${result}
! echo ${result} | grep -E -q "(100% packet loss)|(0 sent)|(no egress interface)"
```

Ping from NSE to NSC:
```bash
result=$(kubectl exec "${NSE}" -n "${NAMESPACE}" -- vppctl ping 2001:db8::1 repeat 4)
echo ${result}
! echo ${result} | grep -E -q "(100% packet loss)|(0 sent)|(no egress interface)"
```

## Cleanup

Delete ns:
```bash
kubectl delete ns ${NAMESPACE}
```
