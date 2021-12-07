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
        - name: rvm-tester
          image: registry.nordix.org/cloud-native/nsm/rvm-tester:latest
          imagePullPolicy: IfNotPresent
          command: ["tail", "-f", "/dev/null"]
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

Start an iperf server in NSC:

```bash
IS_FIRST=$(kubectl exec ${NSCS[0]} -c rvm-tester -n ${NAMESPACE} -- ip a s nsm-1 | grep 172.10.0.1)
if [ -n "$IS_FIRST" ]; then
  kubectl exec ${NSCS[0]} -c rvm-tester -n ${NAMESPACE} -- iperf3 -sD -B 172.10.0.1 -1
  kubectl exec ${NSCS[1]} -c rvm-tester -n ${NAMESPACE} -- iperf3 -i0 t 5 -c 172.10.0.1 -B 172.10.0.2
else
  kubectl exec ${NSCS[1]} -c rvm-tester -n ${NAMESPACE} -- iperf3 -sD -B 172.10.0.1 -1
  kubectl exec ${NSCS[0]} -c rvm-tester -n ${NAMESPACE} -- iperf3 -i0 t 5 -c 172.10.0.1 -B 172.10.0.2
fi
```

Setup a docker container for traffic test:

```bash
docker run --cap-add=NET_ADMIN --rm -d --network bridge-2 --name rvm-tester registry.nordix.org/cloud-native/nsm/rvm-tester:latest tail -f /dev/null
docker exec rvm-tester ip link set eth0 down
docker exec rvm-tester ip link add link eth0 name eth0.100 type vlan id 100
docker exec rvm-tester ip link set eth0 up
docker exec rvm-tester ip addr add 172.10.0.254/24 dev eth0.100
```

Start the client from tester container:

```bash
docker exec rvm-tester ping -c 1 172.10.0.1
```

## Cleanup

Delete the tester container and image:

```bash
docker stop rvm-tester
docker image rm registry.nordix.org/cloud-native/nsm/rvm-tester:latest
true
```

Delete the test namespace:

```bash
kubectl delete ns ${NAMESPACE}
```
