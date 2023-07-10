# Objective 6: Use Consul Service Mesh
**What is a consul service mesh?**
- Provides service to service connection authorization and encryption
    -Uses mTLS for auth and encryption
    - Application can be written for native support using SDK or..
    - use a sidecar proxy (most common)
- Applications may or may not be aware that Consul service mesh is present
    - Traffic between apps flow through the sidecar proxy
    - The proxy enables authenticated and encrypted communications (mTLS) between services
    - Could provide encryption between services that wouldn't otherwise be encrypted
**Primary components of service mesh**
- Service discovery: services come up, register with service discovery and use consul to reach out to other services
- Certificate authority: consul can be a CA and manage CA. Or, outsource to other CAs (including Vault)
- Services: Services running in organization. Registered with service discovery. Communicate acorss other services
- Sidecar proxy: Typically envoy or built-in consul sidecar proxy
- Upstream configuration: Tell consul what services this service needs to reach out to
- mTLS certificates: issued by CA.
- Intentions: Ruleset. Web server can talk to DB. API can talk to identity. Which services  can and can't communicate with each other.

**More Detail**
- CA issues mTLS certificates
    - mTLS is core of consul service mesh
    - consul has built-in CA that can be used
    - Has support to outsource CA functionality to Vault or other solutions
- mTLS certificates
    - Provides authentication by validating the certificate against the CA
    - Enables encryption between the services
- Intentions define access control for Services
    - Determine what services can establish connection to other services
    - Top-down ruleset using Allow or Deny intetions
    - Can be configured via API, CLI  or UI
- Sidecar proxy
    - Service proxy running alongisde the core application
    - Primary sidecar proxy used today is envoy
    - Consul also has buil-in sidecar proxy (not as feature-rich)
    - Other proxies can be used as well
- Service mesh is platform-agnostic
    - Manager across physical networks, public cloud, software defined network, cross-cloud
- Service mesh enables layer 7 observability
    - Proxies see all traffic between services and collect metrics
    - Metrics can be sent to an external monitoring tool, like Prometheus
- Connect must be enabled in the agent configuration (connect stanza)
    - Connect is enabled by default using `-dev` mode
- Upstream vs. Downstream
    - Upstread: The target service that another service depends on.
    - Downstream: The service that is dependent on another target service.

## Objective 6a: Understand Consul Connect service mesh high level architecture
- Database service (mysql), Search service (rails). Both have an envoy proxy running alongside
- Envoy proxy does mTLS encryption and respect allow/deny intentions rulesets.
- Components:
    - Application (Consul service)
    - Sidecar proxy (envoy, consul, something else)
    - mTLS certificate (provided by CA, can be Consul, Vault, something else)
    - Encrypted communication (using mTLS)
    - Intentions: allow/deny ruleset
- Workflow
    - Request connection, service needs to communicate with upstream service
    - Service validates upstream service certificat againast CA bundle
    - Target service validates downstream certificate against the CA bundle
    - Intetion check, target service validates the incoming connection against intentions
    - Connection established, if intention check success, the connection/request is established
- Consul Service mesh - other components
    - L7 traffic management
        - Carve up traffic across the pool of services vs. just using round robin
        - Sometimes called traffic splitting
    - Service mesh gateways
        - Enables routing between federated service mesh datacenters where private connectivity may not be established or feasible
        - Ingress gateways and terminating gateways (k8s)
    - Observability
        - Consul 1.9.0 includes new topology vizualizations to show a service's connectivity.

## Objective 6b: Describe configuration for registering a proxy 
- Just like a service, a sidecar proxy must be registered with Consul
    - Registration does NOT start the sidecar proxy
    - You need to do that on your own and the sidecar listener health check will fail until you do
- Registration is mostly commonly done using the configuration file
    - Usually the same config file as the service using the connect parameter and related options
        ```
        "service": {
            "name": "front-end",
            "port": 8080,
            "connect": {
                "sidecar-service": {}
            }
        }
        ```
        ```
        "service": {
            "name": "front-end",
            "tags": ["v7.05", "production"],
            "address": "",
            "port": 8080,
            "connect": {
                "sidecar-service": {
                    "proxy": {
                        "upstreams": [
                            {
                                "destination_name": "database01"
                            }
                        ]
                    }
                }
            }
        }
        ```
    

## Object 6c: Describe intetions for Consul Connect service mesh
- Intentions describe access control for services
    - Use a service graph to determine what services are allowed to establsh connectiosn to other services
    - Enforced at the destination/target service on inbound connection, proxy requests or within a  natively integrated app (SDK)
- Enforcing Intetions
    - Default behavior is controlled by default ACL policy
        - Allow all means all connections are allowed by default
        - Deny all default means all connections are defined by default
    - Only one intention controls authorization at a given time
- Precedence and match order
    - Top-down ruleset using Allow or Deny intetions
    - Precedence cannot be overwritten
    - https://developer.hashicorp.com/consul/docs/connect/intentions#precedence-and-match-order
- Controlling authorization
    - Authorization is control using either L4 or L7 dpeending on the protocol being used
    - L4 - identity based (TLS) - all or nothing access control based on new connections.
    - L7 - application-aware - can be based on L7 request attributes

## Objective 6d: Check intentions in both the Consul CLI and UI
- Configuring and managing intentions
    - Can be configured using the CLI, API, or UI
    - Intentions can be managed on any interface (i.e intentions created using API can be seen/managed via UI)
    - Changing an intention does not affect existing connections, only new connections
- Intentions are managed primarily using the `service-intentions` config entries or the ui
    - Simpler tasks can be done using the older API or CLI
    - Config
    ```
    # web-01 will be denied connections to db-01 (the upstream service)
    Kind = "service-intentions"
    Name = "db-01"
    Sources: [
        {
        Name = "web-01"
        Action = "deny"
        }
    ]
    ```
    - UI
    - API - PUT v1/connect/intentions/exact, returns turue of intention was created successfully
    ```
    # payload.json
    {
        "SourceType": "consul",
        "Action": "allow"
    }

    curl --request PUT --data @payload.json v1/connect/intentions/exact?source=web-01&destination=db-01
    ```
    - CLI - `consul intention` command
        - get - show information about intention
        - check - validate whether a connection is allowed
        - create - create a new intention (default is allow)
        - delete - delete intention
        - match - show intentions matching a source or destination
        - list - list all intentions
    