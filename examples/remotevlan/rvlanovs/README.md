# NSM Remote Vlan OVS Forwarder

Contains setup for `forwarder-ovs` and device configuration file for remote vlan mechanism.

## Requires

Make sure that you have completed steps from [remotevlan](../../remotevlan) setup.

## Includes

- [Kernel2RVlan](../../use-cases/Kernel2RVlan)

## Run

Deploy the forwarder:

```bash
kubectl apply -k .
```
