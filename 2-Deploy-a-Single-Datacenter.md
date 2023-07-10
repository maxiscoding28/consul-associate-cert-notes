# Objective 2: Deploy a single datacenter
- Consul is typically started using a service manager - e.g. systemctl or Windows Service manager.
- Configuration can be passed as CLI arguments or in configuration file (most common).
- Starting the Consul process using the CLI e.x:
```
consul agent -datacenter="aws" -bind="10.0.10.42" -data-dr=/opt/consul -encrypt=<key> -retry_join="10.0.10.64,10.4.23.98"
```
- **Starting the Consul process using a configuration file:**
  - -config-file
  ```
  consul agent -config-file=/etc/consul.d./config.hcl
  ```
  - -config-dir
  ```
  consul agent -config-dir=/etc/consul.d
  ```
- **Consul Server - Dev Mode**
  - All persistence options are turned off
  - Enables an in-memory server
  - Connect is enabled (will create a new root CA by default)
  - gRPC port default to 8502
  ```
  consul agent -dev
  ```

## Objective 2a: Start and manage the Consul process
- Example systemctl service file:
```
[Unit]
Description="HashiCorp Consul - A service mesh solution"
Documentation=https://www.consul.io/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/consul.d/config.hcl

[Service]
Type=notify
User=consul
Group=consul
ExecStart=/usr/bin/consul agent -config-file=/etc/consul.d/config.hcl
ExecReload=/usr/bin/consul reload
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## Objective 2b: Interpret a Consul agent configuration
- Environment variables cannot be used to configure a Consul client
- Key options in a SERVER configuration file
  - Server: is the server an agent or not?
  - Datacenter: What datacenter to join or create
  - Node: Unique name of agent (usually server name)
  - Join/Retry_join/Auto_join - What other servers/cluster to join
  - client_addr/bind_addr/advertise_addr - interface on node to use for Consul communications


- Example server configuration file
```
log_level  = "INFO"
server     = true
datacenter = "us-east-1"
primary_datacenter = "dc1"

ui_config {
  enabled = true
}

# TLS Configuration
key_file               = "/etc/consul.d/cert.key"
cert_file              = "/etc/consul.d/client.pem"
ca_file                = "/etc/consul.d/chain.pem"
verify_incoming        = true
verify_outgoing        = true
verify_server_hostname = true

# Gossip Encryption - generate key using consul keygen
encrypt                = "pCOEKgL2SYHmDoFJqnolFUTJi7Vy+Qwyry04WIZUupc="

leave_on_terminate = true
data_dir           = "/opt/consul/data"

# Agent Network Configuration
client_addr    = "0.0.0.0"
bind_addr      = "10.0.0.170"
advertise_addr = "10.0.0.170"

# Disable HTTP and use 8501 for HTTPS
ports {
  http  = -1
  https = 8501
}

# Cluster Join - Using Cloud Auto Join
bootstrap_expect = 5
retry_join       = ["provider=aws tag_key=Environment-Name tag_value=consul-cluster region=us-east-1"]

# Enable and Configure Consul ACLs
acl = {
  enabled        = true
  default_policy = "deny"
  down_policy    = "extend-cache"
  tokens = {
    agent = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}

# Set raft multiplier to lowest value (best performance) - 1 is recommended for Production servers
performance = {
    raft_multiplier = 1
}

# Enables auto encrypt for distribution of certs to Consul clients from the Connect CA
auto_encrypt {
  allow_tls = true
}

# Enable service mesh capability for Consul datacenter
connect = {
  enabled = true
}
```

## Objective 2c: Configure Consul network addresses and ports
Use	Default Ports
- DNS: The DNS server (TCP and UDP)	8600
  - Some environments might not be able to send DNS traffic to a non-standard port (UDP 53)
  - Ports below 1024 on linux require to be run with root privileges (We do NOT want consul to be run as root user)
  - We may need to set up forwarding using BIND or dnsmasq to forward requests received on 53 and forward to 8600
- HTTP: The HTTP API (TCP Only)	8500
- HTTPS: The HTTPs API	disabled (8501)*
- gRPC: The gRPC API	disabled (8502)*
- gRPC TLS: The gRPC API with TLS connections	disabled (8503)*
- LAN Serf: The Serf LAN port (TCP and UDP)	8301
- Wan Serf: The Serf WAN port (TCP and UDP)	8302
- server: Server RPC address (TCP Only)	8300
- Sidecar Proxy Min: Inclusive min port number to use for automatically assigned sidecar service registrations.	21000
- Sidecar Proxy Max: Inclusive max port number to use for automatically assigned sidecar service registrations.	21255
- DNS: port 8600 is default port. 

- Consul API
  - `-bind` - interface that Consul agent itself uses
  - `-advertise` - interface that Consul tells other agents and clients to use when connecting to the local 
These are useful to set if server agent has multiple interfaces or if Consul is behind a NAT device

## Objective 2d: Describe and configure agent join and leave behaviors
- Adding Servers
  - Consul servers can join the cluster using multiple methods
  - LAN gossip will propagate membership state across the cluster
  - Agent that is already member of a cluster can join a different cluster
    - Two clusters will be merged into a single cluster
  - CLI - `consul join <host>`
    - Host can be any member of cluster client or server.
    - Not recommended for production deployments
  - Configuration file
    - `-join`
      - Specify one or more agents to join (IPs or hostnames)
      - IF Consul is unable to join specified agents, agent startup fails
    - `-retry_join`
      - Will continue retrying until successful
      - Ideal for automated deployments
    - Cloud Auto-join
      - Use cloud meta-data to discover Consul nodes (tags)
- Removing Servers
  - `consul leave` - triggers a graceful leave and shutdown. 
  - Ensures other nodes see the agent as left rather than failed
  - For servers, a consul leave will reconfigure the cluster to have fewer servers

- Determine the memberships of cluster `consul members`
