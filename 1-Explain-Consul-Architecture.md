# Objective 1: Explain Consul architecture
## Overview: Cloud networking automation for dynamic infrastructure.
- _Service discovery_ - Allows applications to query Consul for available hosts for services.
- _Service segementation_ - Prevent communication between certain services (for security purposes).
- _Service configuration_ - Consul KV store for centralizing configiuration management.

| _OSS_      | _ENT_ |
| ----------- | ----------- |
| Service Discovery | Automated backups |
| Service Segementation | Automated upgrades |
| Layer 7 Traffic Mgmt | Network segments |
| KV storage | Federation | 
| Mesh gateways | Enhanced read scalability | 
| Application aware intentions | Redundancy zones |
|  | Namespaces |
|  | SSO |
|  | Audit logging |

## Why Use Consul?
- Application agnostic _(GO, JS, MongoDB)_
- Platform agnostic _(run on VMware, openshift, K8s)_
- Location agnostic _(AWS, Azure, GCP, on-prem)_

## Problems to solve with Consul:
_Microservices application - Single application is composed of multiple services that can be independently developed, scheduled and scaled. Also introduces operational complexity:
- How do services reliably locate other services that run on ephemeral resources (pods, containers, VMs) with dynamic IP addresses?
- How do we managed network connectivity between these services?
- How do we secure network connectivity between services (preventing and allowing where neccessary)?
- How do we load balance across a service as we scale?
- How do we ensure traffic is only sent to healthy services?

## Objective 1a: Identify the components of Consul datacenter, including agents and communication protocols
### Basic Consul Architecture
**Agent**
- Consul Agent can be run across multiple platforms (Docker, K8s, VM, Physical Server) and OS (Mac, Linux, Windows)
- Typical deployment has a few Servers and many Clients communicating with Servers.
  - Hashicorp recommends 5 server nodes.
- Can be run in Server mode, Client mode, or Dev mode (for testing)
- Server:
  - Consul cluster state
  - Membership
  - Responds to queries
  - Registers services
  - Maintains quorum
  - Acts as Gateway to other Datacenters
- Client:
  - Register local services
  - Perform health checks
  - Forward RPC calls to servers
  - Takes part in LAN Gossip pool
  - Relatively stateless
- Dev 
  - Used only for Testing/Demo
  - Runs as Consul Server
  - Not secure or scalable
  - Runs locally
  - Stores everything in memory
  - Does not write to disk

**Datacenter**
- Single cluster
  - Servers + Clients
- Private
- Low latency, high bandwidth
- Contained in a single location
- Multi-AZ is acceptable
- Uses the LAN gossip pool
  - Used to communicate with all members of the cluster
- Datacenter is **NOT**
  - multi-cloud or location
  - multiple Consul clusters
  - Use the WAN gossip pool
  - Communicate via WAN or internet

**Multi-Datacenter**
- Multi-cloud, multi-region, location or cluster
- Multiple Consul cluster federation
- Uses the WAN gossip pool
- Communicates via WAN or Internet
- Wan federation through mesh gateways

**Consensus Protocol (Raft)**
- Used by the servers for cluster operations
  - Leader elections
  - Maintain log entries across nodes
  - Establishing a quorum
- Key terms:
  - Log:
    - Primary unit of work - an ordered sequence of entries
    - Entries can be a cluster change, KV change etc.
    - All members must agree on the entries and their order to be considered a consistent log
  - Peer set:
    - All members participating in log replication
    - In Consul's case, all servers nodes in the local datacenter
  - Quorum:
    - Majority of members of the peer set (servers)
    - No quorum = no Consul
    - A quorum requires at least (n+1)/2 members
      - 5 nodes requres 3
      - 3 nodes requires 2
- Raft node states:
  - Leader:
    - Ingesting new log entries
    - Processing all queries and transactions
    - Replication to followers
    - Determing when an entry is considered committed
  - Follower:
    - Forwarding RPC requests to the leader
    - Accepting logs from the leader
    - Casting votes for leader election
- Elections/Candidate
  - Leader sends out frequent heartbeats to follower nodes so they know it is active.
  - Each server has a randomly assigned timeout (e.g 150ms - 300ms)
  - If a heartbeast isn't received from the leader, an election takes place
  - The node changes its state to candidate, votes for itself and issues a request for votes to establish majority

**Gossip Protocol (Serf)**
- Used cluster-wide (servers and clients).
- Responsible for:
  - Managing membership of the cluster (clients and servers)
  - Broadcast messages to the cluster such as connectivity failures
  - Allows reliable and fast broadcasts across datacenters
- Uses 2 types of pools.
  - LAN
    - Each datacenter has its own LAN gossip pool
    - Contains all members of the datacenter (clients and servers)
    - Purpose
      - Membership information allows clients to discover servers
      - Failure detection duties are shared by members of entire cluster
      - Reliable and fast event broadcasts
  - WAN
    - Separate globally unique pool
    - All servers participate in the WAN pool regardless of datacenter
    - Purpose
      - Allows servers to perform cross datacenter requests
      - Assists with handling single server or entire datacenter failures

**Networking and Ports**
- All communication happens over HTTP or HTTPS
- Network communication protected by TLS and gossip key
- Ports
  - HTTP API, UI - tcp/8500
  - LAN Gossip - tcp & udp/8301
  - WAN Gossip - tcp & udp/8302
  - RPC - tcp/8300
  - DNS - tcp & udp/8600
  - Sidecard proxy - 21000-21255
- Accessing Consul
  - Consul API can be accessed by any machine (assuming network/firewall)
  - Consul CLI can be accessed and configured from any server node
  - UI can be enable in the configuration file and accessed from anywhere

## Objective 1b: Prepare Consul for high availability and performance
- High Availability is achieved using clustering
  - Hashicorp recommends 3-5 servers in Consul cluster
  - Uses the consensus protocol to establish a cluster leader
  - If a leader becomes unavailable, a new leader is elected
- General recommendation is to not exceed 7 nodes
  - Consul generates a lot of traffic for replication
  - More than 7 servers may be negatively impacted by the network or negatively impact the network

**Consul Enterprise support Enhanced Read Scalability with Read Replicas (Enterprise feature)**
  - Scale your cluster to include read replicas to scale reads
  - Read replicas participate in cluster replication
  - They do not take part in quorum elections (non-voting)
- Voting vs. Non-Voting members
  - Non voting do not participate in the raft quorum
  - Generally used in conjunction with redundancy zones
  - Configured using: `non_voting_member` setting in config file or `-non-voting-member` CLI flag

**Redunandacy Zone (Enterprise feature)**
  - Provides both scaling and resiliency benefits by using non-voting servers
  - Each fault zone only has 1 voting member
    - All others are non-voting members
  - If a voting member fails, a non-voting member in the same fault zone is promoted in order to maintain resiliency and maintain a quorum
  - If an entire zone fails, a non-voting member in a surviving fault zone is promoted to maintain quorum.

**Autopilot (Enterprise feature)**
- Get configuration: `consul operator autopilot get-config`
- Set configuration: `consul operator autopilot set-config -cleanup-dead-servers=false`
- Dead server cleanup
  - Will remove failed servers from cluster once the replacement comes online based on configurable threshold.
  - Cleanup will also be initialized anytime a new server joins the cluster
  - Previosuly, it would take 72 hours to reap a failed server or done manually using `consul force-leave`
- Server stabilization
  - Ensure new Consul node is healthy for x amount of time before being promoted to full voting member.
  - Default time is 10 seconds
- Redundancy zone tags
  - Ensure that Consul voting members will be spread across fault zones to ensure high availability
  - Example: In AWS, you can create fault zones based upon Availability Zones
- Automated Upgrade Migrations
  - New Consul Server Version > current Consul Serversion.
  - Consul won't immediately promote newer servers as voting members
  - Number of new nodes must match number of old nodes before promoting to Voter (and removing old).

## Objective 1c: Identify Consul's core functionality
**Core Feature of Consul**
- Dynamic Service Registration: Applications can register themselves with Consul service when provisioned.
- Service discovery: Applications can use Consul to discover other services across network
- Distributed Health Check: Consul can monitor the health of registered services to ensure traffic is only sent to healthy services.
- KV storage: Store certificates, variables, variables for network configuration.
- ACL: Allow/deny permissions for services and KV data.
- Cross Cloud/Data Center Availability: Service segmentation.
- API/UI/CLI interfaces: Multiple options for managing these features.

#### Service Discovery
- Centralized Service Registry
  - Single point of contact for services to communicate
  - Important for dynamic (ephemeral) workloads (containers).
  - Important for a microservice architecture.
- Reduction or elimination of load balancers to front-end services.
  - Frequently referred to as east/west traffic.
- Real-time health monitoring
  - Distributed responsibility thoughout the cluster
  - Local agent performs query on services
    - Node-level health checks: Is the VM/Container healthy?
    - Application-level health checks: Is the service healthy?
- Automate networking and security using identity-based authorization
  - No more IP-based firewall security (too difficult to manage when scaling)
  - Applications can communicate by providing their service identity.
- Multiple Datacenters
  - Consul can faciliate communication between applications across clouds/on-prem.

#### Service Mesh
- Enable secure communication between services
  - Integrated mTLS communication.
  - Use sidecar architecture placed alongside registered service.
  - Sidecar transparently handles inbound/outbound connections.
- Defined access control for services
  - Defines which service can establish connection to other service.

#### Network Automation
- Dynamic load balancing among services
  - Consul will only send traffic to healthy nodes & services.
  - Use traffic-shaping to influence how traffic is sent.
- Extensible through networking partners
  - F5, Nginx, haproxy, Envoy
- Reduce downtime by using multi-cloud and failover for services
- L7 traffic managed based on workloads and environment.
  - Service failover, path-based routing, and traffic shifting capabilities
- Increased L7 visibility between services
  - View metrics such as connections, timeouts etc.

#### Service Configuration
- Consul provides a distributed KV store.
- All data is replication across all Consul servers.
  - Can be used to store configuration and parameters.
- Can be accessed by any agent (client or server).
  - Accessed using the CLI, API, UI.
  - Make sure to enable ACL to restrict access.
- No restrictions on the type of object stored.
- Primary restriction is the object size - capped at 512kb.
- Doesn't use a directory structure, although you can use / to organize you data in a KV store.
  - This is different than Vault where / signifies a path.

## Objective 1d: Differentiate agent roles
| Server                           | Client                             | Dev                                                   |
|----------------------------------|------------------------------------|-------------------------------------------------------|
| Consul (cluster) State           | Register local services            | Used only for testing/demo                            |
| Membership                       | Performance health checks          | Runs as a consul server                               |
| Response to queries (DNS or API) | Forward RPC calls to servers       | Not secure or scalable                                |
| Register services                | Takes part in LAN gossip pool | Runs locally                                          |
| Maintain quorum                  | Relatively stateless.              | Stores everything in memory and doesn't write to disk |
| Acts as gateway to other DCs     |                                    |                                                       |
