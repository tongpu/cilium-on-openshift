# Cilium deployment on OpenShift

The goal of this repository is to deploy Cilium v1.12.0 using the certified
Cilium operator.

## Create subscription

To deploy Cilium we first need to configure the subscription in the `cilium`
namespace:

```bash
oc apply -f manifests/cilium-olm
```

## Configure Security Context Constraints

The Cilium Helm charts doesn't configure access to the required Security
Context Constraints so we have to add them manually:

```bash
oc adm policy add-scc-to-user hostnetwork system:serviceaccount:cilium:cilium-operator
oc adm policy add-scc-to-user privileged system:serviceaccount:cilium:cilium
```

## Deploy Cilium using CiliumConfig

The configuration is taken verbatim from [`cilium/cilium-olm`](https://github.com/cilium/cilium-olm/blob/b89359b654e689becd116c084a464a2574841a28/ciliumconfig.v1.12.yaml)
and can be deployed using:

```bash
oc apply -f manifests/ciliumconfig.v1.12.yaml
```

