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

Set `multiRegion = true` in the `[Global]` section of the cloud config to let a
single CCM actively manage nodes across multiple AWS regions:

```ini
[Global]
MultiRegion = true
```

When enabled, the CCM determines each node's region from the availability zone
embedded in its provider ID (`aws:///<az>/<instance-id>`) and routes that node's
EC2 calls through an EC2 client for the node's *own* region:

* It keeps using its home-region client for home-region nodes (so single-region
  behavior is unchanged).
* For a node in another region it lazily builds an EC2 client pointed at that
  region's endpoint, caches it, and uses it for `InstanceExists`,
  `InstanceShutdown`, and `InstanceMetadata`. The node is therefore fully
  managed — addresses, instance type, `topology.kubernetes.io/{region,zone}`
  labels, taint removal, and deletion on termination — by the correct region.

Nodes whose region cannot be determined (e.g. a provider ID without an AZ) fall
back to the home-region client, so enabling the flag is safe for single-region
clusters.

### Regional enrichment labels

Region-scoped enrichment that depends on home-region APIs — the zone-ID label
(`DescribeAvailabilityZones`) and network-topology labels — is only applied to
home-region nodes. Foreign-region nodes still receive the core identity fields
(provider ID, addresses, instance type, and the region/zone topology labels
derived from their AZ); they just skip the extra enrichment, which would
otherwise be looked up against the wrong region.

### Enabling without a cloud-config file

Some deployment paths (notably the Helm chart) expose container environment
variables but do not mount an AWS cloud-config file. For those, set the
`AWS_CCM_MULTI_REGION` environment variable instead:

```yaml
env:
  - name: AWS_CCM_MULTI_REGION
    value: "true"
```

The env var only ever turns the behavior **on**; a cloud config that already set
`MultiRegion = true` is never overridden off. An unparseable value (anything
`strconv.ParseBool` rejects) is treated as a fatal configuration error.

## Credentials and IAM

The CCM does **not** need a separate credential per region. AWS IAM is global:
the role the CCM already runs under (e.g. an IRSA role with the standard CCM EC2
permissions) is valid in every region, so the per-region clients all reuse the
CCM's existing credential chain and simply target each region's EC2 endpoint.
Make sure the role's permissions are not constrained to a single region (avoid
region conditions in the policy) and that the regions are reachable from where
the CCM runs.

The only case that needs additional credentials is when the regions live in
**different AWS accounts**. The per-region client construction already threads an
optional assume-role provider through, so cross-account support would be a matter
of supplying a role to assume per account; same-account multi-region needs
nothing beyond the existing role.

## Nodes from other cloud providers

A provider ID's scheme identifies the cloud that owns the node. Anything that is
not the `aws://` scheme (for example `gce://…` or `azure://…`) belongs to a
different provider, and the AWS CCM leaves it completely alone rather than trying
to interpret it as an AWS instance:

* `InstanceExists` reports it as existing, so the node-lifecycle controller never
  deletes another provider's node.
* `InstanceShutdown` reports it as not shutdown.
* `InstanceMetadata` returns an error instead of initializing it.

A bare instance ID with no scheme is still treated as AWS for backwards
compatibility. This guard is independent of `multiRegion`: a non-AWS node is
always left to its own provider.
