# Development

## Basic structure

The network operator consists of a controller loop and a rendering system. The basic flow is:

1. The controller loop detects a configuration change
1. The configuration is preprocessed:
    - validity is checked.
    - unspecified values are defaulted or carried forward from the previous configuration.
    - safety of the proposed change is checked.
1. The configuration is rendered in to a set of kubernetes objects (e.g. DaemonSets).
1. The desired objects are reconciled against the API server, being created or updated as necessary.
1. The applied configuration is stored separately, for later comparison.

### Filling defaults
Because most of the operator's configuration parameters are not changeable, it is important that the applied configuration is stable across upgrades. This has two implications:

**All defaults must be made explicit.**

The network configuration is transformed internally in to a fully-expressed struct. All optional values must have their default set. For example, if the vxlanPort is not specified, the default of 4789 is chosen and applied to the OpenShiftSDNConfig.

Making all defaults explicit makes it possible to prevent unsafe changes when a newer version of the operator changes a default value.

**Some values must be carried forward from the previous configuration.**

Some values are computed at run-time but can never be changed. For example, the MTU of the overlay network is determined from the node's interfaces. Changing this is unsafe, so this must always be carried forward.

Note that the fully-expressed configuration is not persisted back to the apiserver. Instead, it is saved only in the stored applied configuration. An alternative would be to inject these values via a mutating webhook. That requires a running service network, which we don't have until after we've run.

### Validation
Each network provider is responsible for validating their view of the configuration. For example, the OpenShiftSDN provider validates that the vxlanPort is a number between 1 and 65535, that the MTU is sane, etc. No validation is provided via the CRD itself.

## Building

### Test builds
Build binaries manually with
```
./hack/build-go.sh
```

There is a special mode, the renderer, that emulates what the operator _would_ apply, given a fresh cluster. You can execute it manually:

```
_output/linux/amd64/cluster-network-renderer --config sample-config.yaml --out out.yaml
```

### Building images
By default, podman is used to build images. 

```
./hack/build-image.sh
```

You might need sudo:
```
BUILDCMD="sudo podman build" ./hack/build-image.sh
```

Or you could use a docker that supports multi-stage builds
```
BUILDCMD="docker build" ./hack/build-image.sh
```

# Running local builds with the installer

If you want to run a local build with the installer, you need to do a bit of hacking. There is a script that will do most of the dirty work for you. It will modify the installer so the network operator doesn't run, and spin up a local copy of the network operator.

This requires `oc` and `kubectl` in your $PATH.

## Steps

You will need to execute all steps for every installation, since the installer deletes the intermediate files we customize.

1. If you haven't already, build the operator:

```
hack/build-go.sh
```

2. In one window, prepare the cluster:

```
export CLUSTER_DIR=<your cluster directory>
openshift-install --dir=$CLUSTER_DIR create manifests
```

3. In another window, start the operator:

```
export CLUSTER_DIR=<your cluster directory>
hack/run-locally.sh
```

It will print status messages as it waits for the installer to make progress.

4. Optionally, override image references. If you are doing downstream image development, e.g. working on openshift-sdn, you should build, publish, and edit `$CLUSTER_DIR/env.sh` to point to your development image.

5. In the installer window, start the installer:

```
openshift-install --dir=$CLUSTER_DIR create cluster
```

## Tips, Tricks, & Limitations

### Cleaning up
After destroying your cluster, you should delete your `$CLUSTER_DIR`.

If you're using libvirt, you can use `./scripts/maintenance/virsh-cleanup.sh` in the installer repository to quickly delete the virtual machines instead of waiting for Terraform.

### RBAC & CVO
This runs the operator with the `admin` role, so it will have full permissions. When run by the real installer, this will not be the case. Created objects, e.g. DaemonSets, will still have RBAC, though.

In fact, **no** changes from `manifests/` will be picked up except image references.

### Saving install-config between runs
If you're sick of answering all the questions over and over, you can shortcut the installer. Just do
```
openshift-install --dir=$CLUSTER_DIR create install-config
cp $CLUSTER_DIR/install-config.yaml ~/.cache/openshift-install/install-config.yaml
```

Then, for subsequent installer runs, you can do
```
rm -r $CLUSTER_DIR; mkdir -p $CLUSTER_DIR
cp ~/.cache/openshift-install/install-config.yaml $CLUSTER_DIR/
```

### Paring-down installed components
To reduce memory usage, you can remove some non-essential components from the cluster. This will cause some things (most notably, Prometheus) to be skipped. The installer process may or may not succeed, but you should be left with a functional control plane. This is fine for rapid network development. Just set:
```
export HACK_MINIMIZE=1
```

### Developing with a local copy of openshift-sdn

Origin images are difficult to build correctly. If you want to test some binary changes with openshift-sdn, there's an easier way.

#### One-time setup:
1. Set up your API credentials:
    - Log on to https://api.ci.openshift.org/ with your GitHub credentials
    - On the top right, click copy login command. Execute it in a terminal
    - Execute `oc registry login`, which will install credentials for your local podman and docker clients
2. Create the following dockerfile in your origin repo:
```
cat <<EOF > Dockerfile.node-hacking
FROM docker.io/openshift/origin-node:v4.0.0
COPY _output/local/bin/linux/amd64/openshift-sdn /usr/bin/openshift-sdn
EOF
```

#### Building a development sdn image
1.  Pick a registry. You can use the Openshift CI one if you are a member of the organization.
```
export REGISTRY=registry.svc.ci.openshift.org/<YOUR-GITHUB-USERNAME>
```

2. Build the sdn process
```
make WHAT=./cmd/openshift-sdn
```

3. Build and push the node image
```
podman build -t ${REGISTRY}/origin-node:latest -f Dockerfile.node-hacking .
podman push ${REGISTRY}/origin-node:latest
```

4. Follow the steps above, but override the node image reference in step 3.
```
echo "NODE_IMAGE=${REGISTRY}/origin-node:latest" >> ${CLUSTER_DIR}/env.sh
```