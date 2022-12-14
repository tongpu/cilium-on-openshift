# Cilium deployment on OpenShift

Cilium needs to be deployed during the installation of the OpenShift cluster
with `openshift-install` to ensure that it is the only CNI plugin present on
the system. The instructions will guide you through the installation.

The installation guide is based on the [upstream documentation](https://docs.cilium.io/en/v1.11/gettingstarted/k8s-install-openshift-okd/)
so also consult it if anything is unclear.

## Prepare manifests for openshift-install

```bash
# configure variables
CLUSTER_NAME="cilium-demo"
cilium_olm_rev="master"
cilium_version="1.12.0"

# Create install-config.yaml
openshift-install create install-config --dir "${CLUSTER_NAME}"

# Configure Cilium as the networkType
sed -i "s/networkType: .*/networkType: Cilium/" "${CLUSTER_NAME}/install-config.yaml"

# Create manifests
openshift-install create manifests --dir "${CLUSTER_NAME}"

# Copy cilium-olm manifests to the manifests directory
curl --silent --location --fail --show-error "https://github.com/cilium/cilium-olm/archive/${cilium_olm_rev}.tar.gz" --output /tmp/cilium-olm.tgz
tar -C /tmp -xf /tmp/cilium-olm.tgz
cp /tmp/cilium-olm-${cilium_olm_rev}/manifests/cilium.v${cilium_version}/* "${CLUSTER_NAME}/manifests"

# Use CiliumConfig from v1.11, because it contains the correct
# ipam.operator.clusterPoolIPv4PodCIDR configuration
cp -f /tmp/cilium-olm-${cilium_olm_rev}/manifests/cilium.v1.11.7/cluster-network-07-cilium-ciliumconfig.yaml "${CLUSTER_NAME}/manifests"

rm -rf -- /tmp/cilium-olm.tgz "/tmp/cilium-olm-${cilium_olm_rev}"

# Deploy the cluster
openshift-install create cluster --dir "${CLUSTER_NAME}"
```

## Configure Cilium using CiliumConfig

A basic Cilium configuration is already deployed using the manifests, but any
further configuration can now be applied using the CiliumConfig CRD.

To enable Hubble and the Hubble UI we need to patch the `cilium-olm` role
to also manage Ingress resources. Additionally you need to make sure that
TCP/4244 is opened on each node from within the cluster. This is required for
Hubble Relay to connect to each node.

```bash
oc patch -n cilium role cilium-olm --type='json' --patch='[{"op": "add", "path": "/rules/-", "value": {"apiGroups": ["networking.k8s.io"], "resources": ["ingresses"], "verbs": ["*"]}}]'
oc apply -f manifests/cilium/ciliumconfig.v1.12.yaml
```

## Install OpenShift Distributed Tracing

We're deploying the OpenShift Distributed Tracing operators according to the
[Red Hat documentation](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.11/html/distributed_tracing/distributed-tracing-installation).

```bash
oc apply -f manifests/openshift-distributed-tracing
```

And after that we're deploying Jaeger and OpenTelemetry Collector. Because of
the way the OpenTelemetry Operator is configuring the labels on the pod and
selector of the services I had to upload the `hubble-otel` image (`ghcr.io/cilium/hubble-otel/otelcol:v0.1.1`)
to Docker Hub with the label `latest` to ensure that they match up.

```bash
oc apply -f manifests/openshift-distributed-tracing
```

## Deploy demo application

This will deploy the OpenTelemetry demo application to OpenShift. It will send
its traces to the OpenTelemtry Collector we've deployed before, where they're
going to be correlated to the Hubble traces.

```
oc create namespace cilium-demo
oc adm policy add-scc-to-user anyuid system:serviceaccount:cilium-demo:default
oc kustomize --enable-helm | oc apply -f -
```
