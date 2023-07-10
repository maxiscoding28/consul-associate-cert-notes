# 	Objective 4: Access the Consul key/value (KV)

## Objective 4a: Understand the capabilities and limitations of the KV store
- Access via API, CLI, or UI (rare)
- To limit access to the KV, use Consul ACLs (objective 8)
- Consul KV data is only replicated across server nodes
- Responses are Base64 encoded
- Accessed by Server or Clients or locally with valid ACL token
- consul snapshot save for backup and recovery
- Object size limit 512kb


## Objective 4b: Interact with the KV store using both the Consul CLI and UI
- Adding data to consul with a PUT `consul kv put`
- Reading data `consul kv get`
- Delete `consul kv delete`
- Export
- Import

## Object 4c: Monitor KV changes using watch
- Watch provides a way to monitor for specific changes in Consul
    - Buil-in to Consul
    - Once the view of data is updated, a specific handler is invoked
    - Or just log to STDOUT
    - Handlers can invoke a shell command or hit HTTP endpoint when change is detected 

- Typoes of watchers: 
    - key: watches a specific KV pair
    - keyprefix: watch a prefix in KV store
    - services: Watch list of available services
    - nodes: Watch list of nodes
    - service: Watch instances of a service
    - checks: Watch health check
    - event: Watch for custom user event

- Watches are implemented using blocking queries in Consul API
- Watches can be added to agent configuration
- It can be started outside of agent using `consul watch`
- Config
```
# when this key changes, run script to update creds in application
{
    "type": "key",
    "key": "prod/database/mysql",
    "handler_type": "script",
    "args": [/usr/bin/update_creds.sh]
}
```

Can also use cli
`consul watch -type=key -key=prod/database/mysql /usr/bin/update_database.sh`



## Objective 4d: Monitor KV changes using envconsul and consul-template
**envconsul**
- Envconsul launches a subprocess to set env variables from data retrieved from Consul (and Vault)
- Separate binary that runs on the application server (consul client)
- envconsul populates the ENV and the application reads the ENV
    - Apps no longer need to read config files with sensitive data in clear text
    - Retrieve data from the KV or about Consul services
- Allows simplified application integration without the application knowing it's using Consul to retrieve values
```
consul kv put db/DB_ADDR 10.2.23.98
consul kv put db/PORT 3306

envconsul -prefix db env

# new env variables
DB_ADDR=10.2.23.98
PORT=3306
```

Workflow
- Container scheduled
- Envconsul executed
    - KV get
    - Response
- Env variables populated
- Application launched

**consul-template**
- consul template populates values from Consul into the file system running the consul-template daemon
- Separate binary that runs on the application server (Consul client)
- Uses a preconfigured, templated file as an input
- Output a file with data populated from Consul

Workflow:
- VM launched
- Consul-template executed
    - KV get
    - Retrieves response
- Config file created
- Application launch
