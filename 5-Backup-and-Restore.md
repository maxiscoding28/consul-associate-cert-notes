#  Objective 5: Backup and Restore
- Consul snapshots are point in time snapshots of Consu lstate
- Snapshot is the primary backup and DR solution for Consul

- By default, snapshots are taken in consistent mode, meaning leader performs the snapshot
    - The leader validates whether it is the leader first
    - A follower can take the snapshot if the `-stale` flag is used
        - Usefult to reduce the load on a leader but could lose data
        - Also useful if a cluster does not have a leader

## Objective 5a: Describe the contents of a snapshot
- Snapshots are gzipped tar archive that includes but not limited to:
    - KV entries
    - Service catalog
    - Prepared queries
    - Sessions
    ACLs

## Objective 5b: Back up and restore the datacenter
**backup**
- Snapshots can be taken using API or CLI
- They can be manually taken or automated by external process
- [Enterprise] use Consul Snapshot Agent
- Requires a valid ACL token to perform
- Manual snapshots could be taken before:
    - Consul upgrades - provides a way to fail back
    - Bootstrap a new identical datacenter with the same name

**restore**
- Restoring consul from snapshot is usually done when recoverying from a DR scenario
- A restore is a disruptive process and it is an all or nothing action
    - You cannot selectively restore data
- Restoring Consul is also not designed to handle a server failure during the restore process

**commands**
```
consul snpashot
    agent - run the agent as a long-running daemon
    inspect - view metadata abotu an existing snapshot file
    restore - restore the referenced snapshot to Consul
    save - create a new snapshot
```

- If using acls, you need a token:
```
export CONSUL_HTTP_TOKEN=abc-abc-abc-abc
consul snpashot save backup.snap
```

## Object 5c: 	[Enterprise] Describe the benefits of snapshot agent features
Enterprise only

- Long running daemon that regularly takes snapshots of the Consul cluster
- Customizable interval (how frequently take snapshots)
- Retention configuration (how many snapshots to keep)
- Multiple options to store snapshots
    - Local
    - S3
    - Azure Blob
    - GCP Blob

- Benefits
    - Automated snapshots of the cluster
    - Manages its own leadership election for high availability (can be run on every node, but snap will be from leader)
    - Provides failover in event the leader becomes avaiable
    - Registers itself as a Consul service
        - Easy to keep track of status and health using API, UI, CLI
        - Health checks can alert you of problems so you can take action.

Workflow
- Create consul snapshot agent configuration file (`/etc/snapshot.d/snapshot.json`)
- Configure service on underling OS (`/etc/systemd/system/snapshot.service`) 
- Start the service (`systemctl start snapshot`)

Example configuration file
```
{
   "snapshot_agent": {
      "http_addr": "127.0.0.1:8500",
      "datacenter": "",
      "token": "<insert ACL token here>",
      "snapshot": {
         "interval": "30m",
         "retain": 336,
         "deregister_after": "8h",
         "service": "consul-snapshot"
      },
      "aws_storage": {
        "s3_region": "us-east-1",
        "s3_bucket": "xxx-xxx-consul-snapshots"
      }
   }
 }

```