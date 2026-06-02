# Multi-region node support

By default the AWS cloud-controller-manager (CCM) assumes every node in the
cluster lives in the CCM's own region. It only constructs an EC2 client for that
one region (taken from the `[Global] Region` config value, or discovered from
IMDS on the node it runs on).

This becomes a problem when a single Kubernetes cluster has nodes in more than
one AWS region — for example, worker nodes in a second region that join the
control plane over VPC peering. For those out-of-region nodes:

* The node-lifecycle controller periodically calls EC2 `DescribeInstances` for
  every node's backing instance. Because the instance does not exist in the
  CCM's region, EC2 returns `InvalidInstanceID.NotFound`, which the CCM
  interprets as "the instance is gone" and **deletes the otherwise-healthy
  Node**.
* The node controller cannot describe the instance either, so it cannot
  initialize the node (provider ID, addresses, topology labels, taint removal).

## The `multiRegion` option

Set `multiRegion = true` in the `[Global]` section of the cloud config to make
the CCM region-aware:

```ini
[Global]
MultiRegion = true
```

When enabled, the CCM determines each node's region from the availability zone
embedded in its provider ID (`aws:///<az>/<instance-id>`). Nodes whose region
differs from the CCM's own region are left untouched:

* `InstanceExists` reports them as existing, so the node-lifecycle controller
  does **not** delete them.
* `InstanceShutdown` reports them as not shutdown.
* `InstanceMetadata` does not attempt to initialize them.

Nodes whose region matches the CCM's region (or whose region cannot be
determined, e.g. a provider ID without an AZ) are handled exactly as before, so
enabling the flag is safe for single-region clusters.

## Operating model

`multiRegion` makes the CCM ignore nodes outside its region; it does not make a
single CCM manage instances across regions. Out-of-region nodes must therefore
get their identity and lifecycle from somewhere else:

* **Initialization** — register the out-of-region nodes with their provider ID
  and topology labels set at the kubelet level (e.g. `--provider-id` and
  `--node-labels`) so they don't need the CCM to initialize them, or run a CCM
  in that region that owns those nodes.
* **Deletion on termination** — because this CCM no longer reaps out-of-region
  nodes when their instances terminate, that responsibility moves to the
  region's own CCM or to external lifecycle management (e.g. an ASG lifecycle
  hook / termination handler, or AWS Node Termination Handler).

This makes the standard "one CCM per region, each owning its region's nodes"
topology safe: each CCM ignores the other regions' nodes instead of deleting
them.
