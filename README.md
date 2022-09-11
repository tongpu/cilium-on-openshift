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

## Deploy demo application

This will deploy the example microservices-demo application to OpenShift:

```
oc create namespace cilium-demo
oc label namespace cilium-demo openshift.io/cluster-logging=true openshift.io/cluster-monitoring=true
oc adm policy add-scc-to-user anyuid system:serviceaccount:cilium-demo:default
oc -n cilium-demo apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.3.9/release/kubernetes-manifests.yaml
oc -n cilium-demo delete service frontend-external
oc -n cilium-demo apply -f manifests/cilium-demo/frontend-ingress.yaml
```
