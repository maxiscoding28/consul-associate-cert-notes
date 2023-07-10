# Objective 3: Register services and use service discovery
- **What is a service?**: Multiple definitions. Service is any workload or application that an organization is running that is providing some services to users (either internal or external)

- **Service Discovery:** Put all these services in a service catalog. Users can make DNS query to find services. Other application can make API request to find services.

## Objective 3a: Interpret a service registration
### Creating a Service Definition file
Example:
```
service {
    # Name of the node (server/container)
    # Needs to be unique within the service
    id = "web-server-01"
    
    # Name of the service to be register
    # Must follow DNS standards. Pre-pended to DNS name/api endpoint for 
    # service discovery
    name = "front-end-eCommerce"
    
    # optional tags for service
    tags = ["prod", "webapp"]
    
    # IP address/interface for the service
    address = "10.0.101.110"
    
    # Port service is running on
    port = 80

    # optional health check for the service
    check = {
        id = "web"
        name = "Check web on port 80"
        tcp = "localhost:80"
        interval = "10s"
        timeout = "1s"
    }
}
```
- Defaults:
    - ID will be set to the Name if not set
    - Address will be set to the default address of Consul agent
- Default namespace for a registered service
    - <name>.service.consul
    - `front-end-eCommerce.service.consul`
- Multiple nodes in the catalog providing the same service
    - Provides high availability
    - Only registered services passing health checks will be returned


## Objective 3b: Differentiate ways to register a single service
- How do I register a service in Consul?
    - Register with local agent using a service definition file or HTTP API

- Service Registration typically happens when a new service is provisioned
    - Container is schedule by K8s
    - Instance is deploy via Terraform
    - Jenkins provions new VMs on VMware cluster

- HTTP API: PUT /v1/agent/service/register { "service": { "name": "retail-web", "port": 8080 } }

Service definition file (.hcl or .json)
    - Create a single file and set using the `-config-file` parameter.
    - Place file inside `-config-dir` (read at startup).
    - Run the `consul services register CLI`.
    - Execure a consul reload command after adding the file.

## Objective 3c: Interpret a service configuration with health check
### Configure a Service Health Check
- Determine if service is healthy
- Can be created via API or service configuration
- Health check configuration may include
    - name
    - arguments
    - interval (how often it runs)
    - Additional params based on health check
- Types of health checks:
    - Application-level for the service itself
    - System-level for the node
    - Script health check
    - HTTP health check
    - TCP health check
    - TTL health check
    - Docker health check
    - gRPC health check
    - Alias healt check
- Service may have multiple health checks defined
```
    check = {
        id = "web"
        name = "Check web on port 80"
        tcp = "localhost:80"
        interval = "10s"
        timeout = "1s"
    }
```

## Objective 3d: Check the service catalog status from the output of the DNS/API interface or via the Consul UI
- DNS query - most commonly used
    - 1. DNS request from App
    - 2. DNS forwarder to forward requests to consul (anything wiht `.consul`)
    - 3. Consul returns healthy nodes
    - 4. Responds to DNS query
    - 5. App connects to service..
    - Example: `dig @127.0.0.1 -p 8600 front-end-eCommerce.service.consul`
- API request - requires app integration
    - Example: `curl http://127.0.0.1:8500/v1/catalog/service/front-end-eCommerce | jq`
- Consul UI - least commonly used

## Objective 3e: Interpret a prepared query
- Example: "I need to connect to a service, it must be healthy, it must have web-app, it must have tag v6.4"
    - Prepared query:
    ```
{
  "Name": "web-app-v64",
  "Service": {
    "Service": "web-app",
    "Tags": ["v6.4"]
  }
}
    ```
    - Execure prepared query DNS - `web-app-v64.query.consul`
    - https:consul.example:8500/v1/query/execute

### Adding failover policies
- When multiple datacenters are federated, we can extend prepared queries to return services in other datacenters
- Extension of prepared queries
- Transparent to application
- Determines target for a service request

### Types of failover policies
- Static policy - fixed list of the order of failover. Try dc1. Then try dc2.
```
{
  "Name": "web-app-v64",
  "Service": {
    "Service": "web-app",
    "Tags": ["v6.4"]
    "Failover": {
        "Datacenters": ["dc2", "dc3"]
    }
  }
}

**avaialble at web-app-v64.query.consul**
```
- Dynamic policy - send to nearest DC based on rount trip time (RTT). Try 2 DC and send to quickest RTT.
```
{
  "Name": "web-app-v64",
  "Service": {
    "Service": "web-app",
    "Tags": ["v6.4"]
    "Failover": {
        "NearestN": 2
    }
  }
}
```
- Hybrid policy - use shortest RTT first, then use other DCs. Try 2 DC and sent to quickest RTT. If that fails, try DC1 then DC2. Each datacenter is only queried one time in a failover
```
{
  "Name": "web-app-v64",
  "Service": {
    "Service": "web-app",
    "Tags": ["v6.4"]
    "Failover": {
        "NearestN": 2
        "Datacenters": ["dc2", "dc3"]
    }
  }
}
```
- Failover policies will try to return a local service first before returning a service from a federated datacenter

## Objective 3f: Use a prepared query
```
cat > /home/ec2-user/prep-query.json << EOF
{
    "Name": "eCommercev8",
    "Service": {
        "Service": "front-end-eCommerce",
        "Tags": [ "v8", "production" ]
    }
}
EOF

curl --request POST --data @/home/ec2-user/prep-query.json http://127.0.0.1:8500/v1/query

dig @127.0.0.1 -p 8600 eCommercev8.query.consul
```
