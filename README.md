# Cilium deployment on OpenShift

The goal of this repository is to deploy Cilium v1.12.0 using the certified
Cilium operator.

## Create Cilium subscription

To deploy Cilium we first need to configure the subscription in the `cilium`
namespace:

```bash
oc apply -f manifests/cilium-olm
```

## Deploy Cilium using CiliumConfig

The Cilium configuration is based on the the configuration from
[`cilium/cilium-olm`](https://github.com/cilium/cilium-olm/blob/b89359b654e689becd116c084a464a2574841a28/ciliumconfig.v1.12.yaml)
and additionally some ClusterRoleBindings are required to grant the
ServiceAccounts used by Cilium access to required SecurityContextConstraints,
because they are not managed by the Cilium Helm chart itself:

```bash
oc apply -f manifests/cilium
```

