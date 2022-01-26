# Test kernel to remote vlan connection

This example shows that NSCs can connect to a cluster external entity by a VLAN interface. There is two vlan network configured.
NSCs are using the `kernel` mechanism to connect to local forwarder.
Forwarders are using the `vlan` remote mechanism to set up the VLAN interface.

## Requires

Make sure that you have completed steps from [remotevlan](../../remotevlan) setup.

## Run

Create test namespace:

```bash
NAMESPACE=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/7dde61822c0256ccb4aef4e4c33d4fb9341a0296/examples/use-cases/namespace.yaml)[0])
FIRST_NAMESPACE=${NAMESPACE:10}
NAMESPACE=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/7dde61822c0256ccb4aef4e4c33d4fb9341a0296/examples/use-cases/namespace.yaml)[0])
SECOND_NAMESPACE=${NAMESPACE:10}
```

Create first iperf server deployment:

```bash
cat > first-iperf-s.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iperf1-s
  labels:
    app: iperf1-s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iperf1-s
  template:
    metadata:
      labels:
        app: iperf1-s
      annotations:
        networkservicemesh.io: kernel://finance-bridge/nsm-1
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - iperf1-s
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: iperf-server
        image: networkstatic/iperf3:latest
        imagePullPolicy: IfNotPresent
        command: ["tail", "-f", "/dev/null"]
EOF
```

Create second iperf server deployment:

```bash
cat > second-iperf-s.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iperf2-s
  labels:
    app: iperf2-s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iperf2-s
  template:
    metadata:
      labels:
        app: iperf2-s
      annotations:
        networkservicemesh.io: kernel://private-bridge/nsm-1
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - iperf2-s
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: iperf-server
        image: networkstatic/iperf3:latest
        imagePullPolicy: IfNotPresent
        command: ["tail", "-f", "/dev/null"]
EOF
```

Deploy the iperf-NSCs:

```bash
kubectl apply -n ${FIRST_NAMESPACE} -f ./first-iperf-s.yaml
kubectl apply -n ${SECOND_NAMESPACE} -f ./second-iperf-s.yaml
```

Wait for applications ready:

```bash
kubectl -n ${FIRST_NAMESPACE} wait --for=condition=ready --timeout=1m pod -l app=iperf1-s
kubectl -n ${SECOND_NAMESPACE} wait --for=condition=ready --timeout=1m pod -l app=iperf2-s
```

Get the iperf-NSC pods:

```bash
IPERF1_NSCS=($(kubectl get pods -l app=iperf1-s -n ${FIRST_NAMESPACE} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'))
IPERF2_NSCS=($(kubectl get pods -l app=iperf2-s -n ${SECOND_NAMESPACE} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'))
```

Create a docker image for test external connections:

```bash
cat > Dockerfile <<EOF
FROM networkstatic/iperf3

RUN apt-get update \
    && apt-get install -y ethtool tcpdump \
    && rm -rf /var/lib/apt/lists/*

ENTRYPOINT [ "tail", "-f", "/dev/null" ]
EOF
docker build . -t rvm-tester
```

Setup a docker container for traffic test:

```bash
docker run --cap-add=NET_ADMIN --rm -d --network bridge-2 --name rvm-tester rvm-tester tail -f /dev/null
docker exec rvm-tester ip link set eth0 down
docker exec rvm-tester ip link add link eth0 name eth0.100 type vlan id 100
docker exec rvm-tester ip link add link eth0 name eth0.200 type vlan id 200
docker exec rvm-tester ip link set eth0 up
docker exec rvm-tester ip addr add 172.10.0.254/24 dev eth0.100
docker exec rvm-tester ip addr add 172.10.0.253/24 dev eth0.200
docker exec rvm-tester ethtool -K eth0 tx off
```

Check first vlan from tester container:

```bash
status=0
for nsc in "${IPERF1_NSCS[@]}"
do
  IP_ADDR=$(kubectl exec ${nsc} -c iperf-server -n ${FIRST_NAMESPACE} -- ip -4 addr show nsm-1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
  docker exec rvm-tester ping -c 1 ${IP_ADDR} -I eth0.100
  if test $? -eq 1 
    then
      status=1
  fi
done
if test ${status} -eq 1
  then
    false
fi
```

Check second vlan from tester container:

```bash
status=0
for nsc in "${IPERF2_NSCS[@]}"
do
  IP_ADDR=$(kubectl exec ${nsc} -c iperf-server -n ${SECOND_NAMESPACE} -- ip -4 addr show nsm-1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
  docker exec rvm-tester ping -c 1 ${IP_ADDR} -I eth0.200
  if test $? -eq 1 
    then
      status=1
  fi
done
if test ${status} -eq 1
  then
    false
fi
```

## Cleanup

Delete the tester container and image:

```bash
docker stop rvm-tester
docker image rm rvm-tester:latest
true
```

Delete the test namespace:

```bash
kubectl delete ns ${FIRST_NAMESPACE}
kubectl delete ns ${SECOND_NAMESPACE}

```
