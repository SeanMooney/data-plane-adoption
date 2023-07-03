Node Selector
=============

There are a variety of reasons why we may want to restrict the nodes where
OpenStack services can be placed:

- Hardware requirements: System memory, Disk space, Cores, HBAs
- Limit the impact of the OpenStack services on other OpenShift workloads.
- Avoid collocating OpenStack services.

The mechanism provided by the OpenStack operators to achieve this is through the
use of labels.

We would either label the OpenShift nodes or use existing labels they already
have, and then use those labels in the OpenStack manifests in the
`nodeSelector` field.

The `nodeSelector` field in the OpenStack manifests follows the standard
OpenShift `nodeSelector` field, please refer to [the OpenShift documentation on
the matter](https://docs.openshift.com/container-platform/4.13/nodes/scheduling/nodes-scheduler-node-selectors.html)
additional information.

This field is present at all the different levels of the OpenStack manifests:

- Deployment: The `OpenStackControlPlane` object.
- Component: For example the `cinder` element in the `OpenStackControlPlane`.
- Service: For example the `cinderVolume` element within the `cinder` element
  in the `OpenStackControlPlane`.

This allows a fine grained control of the placement of the OpenStack services
with minimal repetition.

Values of the `nodeSelector` are propagated to the next levels unless they are
overwritten. This means that a `nodeSelector` value at the deployment level will
affect all the OpenStack services.

For example we can add label `type: openstack` to any 3 OpenShift nodes:

```bash
$ oc label nodes worker0 type=openstack
$ oc label nodes worker1 type=openstack
$ oc label nodes worker2 type=openstack
```

And then in our `OpenStackControlPlane` we can use the label to place all the
services in those 3 nodes:

```
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
spec:
  secret: osp-secret
  storageClass: local-storage
  nodeSelector:
    type: openstack
< . . . >
```

What if we don't mind where any OpenStack services go but the cinder volume and
backup services because we are using FC and we only have HBAs on a subset of
nodes? Then we can just use the selector on for those specific services, which
for the sake of this example we'll assume they have the label `fc_card: true`:

```
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
spec:
  secret: osp-secret
  storageClass: local-storage
  cinder:
    template:
      cinderVolumes:
          pure_fc:
            nodeSelector:
              fc_card: true
< . . . >
          lvm-iscsi:
            nodeSelector:
              fc_card: true
< . . . >
      cinderBackup:
          nodeSelector:
            fc_card: true
< . . . >
```

The Cinder operator does not currently have the possibility of defining
the `nodeSelector` in `cinderVolumes`, so we need to specify it on each of the
backends.

It's possible to leverage labels added by [the node feature discovery
operator](https://docs.openshift.com/container-platform/4.13/hardware_enablement/psap-node-feature-discovery-operator.html)
to place OpenStack services.

## MachineConfig

Some services require us to have services or kernel modules running on the hosts
where they run, for example `iscsid` or `multipathd` daemons, or the
`nvme-fabrics` kernel module.

For those cases we'll use `MachineConfig` manifests, and if we are restricting
the nodes we are placing the OpenStack services using the `nodeSelector` then
we'll also want to limit where the `MachineConfig` is applied.

To define where the `MachineConfig` can be applied we'll need to use a
`MachineConfigPool` that links the `MachineConfig` to the nodes.

For example to be able to limit `MachineConfig` to the 3 OpenShift nodes we
marked with the `type: openstack` label we would create the
`MachineConfigPool` like this:

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: openstack
spec:
  machineConfigSelector:
    matchLabels:
      machineconfiguration.openshift.io/role: openstack
  nodeSelector:
    matchLabels:
      type: openstack
```

And then we could use it in the `MachineConfig`:

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: openstack
< . . . >
```

Refer to the [OpenShift documentation for additional information on `MachineConfig` and `MachineConfigPools`](https://docs.openshift.com/container-platform/4.13/post_installation_configuration/machine-configuration-tasks.html)

**WARNING:** Applying a `MachineConfig` to an OpenShift node will make the node
reboot.
