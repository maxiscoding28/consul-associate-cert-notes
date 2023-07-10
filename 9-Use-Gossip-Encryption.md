## Objective 9: Use Gossip Encryption

### Objective 9a: Understanding the Consul security/threat model
Security model review
- Gossip protocol encryption
- Built-in ACL system
- Consul Agent Communication
- mTLS for Authenticity and Encryption
- Certificate Authority

### Objective 9b: Configure gossip encryption for the existing datacenter
- Gossip protocol encryption
    - Gossip protocol uses a symmetric key
    - Essentially a shared secret method for both servers and clients in a cluster
    - The same key must be used across all agents in the cluster
    - Agents in any federated datacenters must ALSO use the same key
- Encryption Key
    - 32-byte, Base64 encoded key used for encryption
    - You can use the built-in tool `consul keygen` to generate a new key
    - Can use `consul keyring` to list and manage keys in Consul
- Gossip Encryption
    - By default, gossip encryption is not enabled, messages are sent in clear-text
    - Gossip key is added to the agent configuration using the `encrypt` parameter
    - Can also specify the key using the `-encrypt` flag if launching from the CLI
    -  Encryption key needs to be added to servers _and_ clients
- Configure Existing Cluster with Gossip Encryption
    - You can modify a cluster to enable gossip encryption without incurring Consul downtime
    - Does require rolling restarts of Consul agents (`consul reload` will not work here)
    - Requires the addition of two parameters
        - `encrypt_verify_incoming` - used to disable the encryption for incoming gossip
        - `encrypt_verify_outgoing`- used to disable the encryption for outgoing gossip
    - Steps:
        - 1. Generate a new gossip encryption key `consul keygen`
        - 2. Add new key parameters in the agent config file
        ```
        encrypt: $KEY
        encrypt_verify_incoming = false
        encrypt_verify_outgoing = false
        ```
        - 3. Perform rolling update of all agents `systemctl restart consul`
        - 4. Set `encrypt_verify_outgoing` as 'true' on all agents
        - 5. Perform rolling update of all agents `systemctl restart consul`
        - 6. Set `encrypt_verify_incoming` as 'true' on all agents
        - 7. Perform rolling update of all agents `systemctl restart consul`

### Objective 9c: Manage the lifecycle of encryption keys
- After initial deployment, use consul keyring to manage encryption keys
    - List existing keys
    - Distribute new keys
    - Retire old keys
    - Change the key used for encryption
- You still use `consul keygen` to create keys
    - Consul keyring doesn't generate new keys
- There can be multiple keys at any given time but it's not recommended
    - Becomes more expensive with multiple keys, since Consul would require multiple attempts to decrypt the key. Recommended to have multiple keys only during rotation operations.
```
consul keygen

consul keyring -list

consul keyring -install <KEY>

consul keyring -remo
```
