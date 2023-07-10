## Objective 8: Secure services with Basic ACLs
**Introduction to the Consul ACL system**
- Consul ACL system is built-in but optional feature of Consul
    - Controls access to data and the Consul API
    - Relies on tokens that are associated with policies which define access
- Key components of the ACL system
    - Basic:
        - Token - a bearer token used during the UI, CLI, API request
        - Policy - a grouping of rules that determine the fine-grained rules to be applied to a token
    - Advanced:
        - Roles - group a set of policies and service identities applied to many tokens
        - Service identities - policy template to link a policy to a token or role

- More about Service Identities
    - An ACL policy template that links a policy for services in Service mesh (connect)
    - Helps avoid boilerplate policy creation since similar policies tend to look identical when you have many services registered and using the service mesh feature
    - Helps services/sidecards to be discovered and easy discover other services
    - Can be used on tokens and roles
    - Applies preconfigured ACL rules
- Service Identities are composed of:
    - Service - the name of the service (and possibly sidecar proxy)
    - Datacenters - list of datacenters that policy is valid for

- More about roles
    - A named set of policies and service identities
    - Sort of a grouping of multiple policies and service identities that can be assigned to many tokens
- Roles are composed of:
    - ID - auto generated
    - Name - unique within consul
    - Description - human readable
    - Policy set - a list of policies to apply to the role
    - Service identities - a list of service identities for the role
### Set up and configure a basic ACL system
- Enable the Consul ACL system
    - By default is not enabled
    - ACLs are eenable in the agent config file for Consul servers and clients
    - Configuration parameters include default policy and other parameters
    ```
    "acl" {
        "enabled": true,
        "default_policy": "deny",
        "down_policy": "extend-cache",
        "tokens": {
            "agent": "abcdefghijk"
        }
    }
    ```
- Bootstrapping the ACL system
    - Required administrative action before ACL system can be used
    - Usually done one time during inital configuration
    - Default_policy should be set to Allow during bootstrapping and configuration phase
    - Bootstrapping create the bootstrap/master token and the anonymous token
    - Creates the global management policy

Workflow:
- Provision Consul cluster
- Bootstrap the ACL system
- Create policies
- Create tokens
- Configure Agents/Clients with token
- Update default policy to deny

### Create policies
- Policy Control Level Permissions
    - Read: Allow the resouce to be read
    - Write: Allow the resource to be read and modified
    - Deny: Do not allow any permissions to resource
    - List: Allow access to all keys under a segment in the Consul K/V
- ACL Resources Available for Rules
    - ACL - operations and managing the ACL
    - Agent - Utility operations in the Agent API
    - Event - Listing and firing events in the Events API
    - Key - K/V store operations
    - Keyring - Keyring operations
    - Node - Node-level catalog operations
    - Operator - Cluster level operations
    - Query - Prepared query operations
    - Service - Service level catalog operations
    - Session - session operations
- Examples:
    - Exact name
    - Prefix
    -
- Consul acl policy
    - create, read, delete, list, update,
- Anonymous token
    - Request is made to consul that does not include anonymous token
    - Actions in mind that you want unauthenticated clients in consul to do
        - Query services for IP/Hosts
        - Read a prepared query
        - Maybe no token to run consul members
- Recomendation
    - Create one policy per node
```
node "i-06915819567ba85fe" {
    policy = "write"
}
key_prefix "vault/" {
    policy = "write
}
key_prefix "vault/" {
    policy = "read
}
```
`consul acl policy create -name "Test" -rules @policy.hcl -token a67275e2-607e-c27c-5654-ebd0e29f8548`
- Need root token in order to perform initial ACL bootstrapping operations

### Manage token lifecycle: multiple policies, token revoking, ACL roles, service identities
- Core Components of Consul ACLS
- Tokens
    - Accessor: Name or ID of the token
    - Tokens are created and attached to the policy
    - Used to determine if the request is authorized to perform the requested actions
- Basic components of a token include:
    - Accessor
    - Secret ID
    - Policy Set
    - Description

- Default ACL token
- Bootstrap/Master token
    - Has unrestricted privilges (linked to Global Management policy)
    - Initial method for authentication of ACL configuration
    - Bootstrap should not be used on a day to day basis and security protected
    - If the bootstrap token is lost, there is a reset to recreate one
    - Secret ID will be unique
- Anonymous token
    - Used when a request is made that does not specify a bearer token
    - Cannot delete the token but you can update the description and privileges
    - Commonly set to read services (DNS/API) for unauthenticated clients (possibly more)
    - ID is always 0000-00000...2
- Additioanl token attributes
    - Expiration time
        - Optional configuration to set time when token is revoked
        - Duration fo time (i.e. 30m, 24h, 3d)
    - Roles
        - When tokens are created, they can be assigned a pre-configured role that will be used for the token
        - Can specify role name or role id
    - Service identities
        - Tokens can also be assigned to one or more service identities
- Token needs for production
    - Consul server nodes: nodes need to communicate within the cluster
    - Consul clients - need access to speciffic services and/or KV
    - Consul snapshot agent - need access to take snapshots
    - Consul Administrative Tasks - any task, including consul members will require a token


### Perform an API request using a token

```
-token

-token-file

-H X-Consul-Token
```
