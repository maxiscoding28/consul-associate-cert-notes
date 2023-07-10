## 	Objective 7: Secure Agent Communication
- Consul is not secure (by default)
    - Security features of consul need to be configured to secure. Example - certificates/TLS need to be set up

### Objective 7a: Understanding Consul security/threat model
**Security Model**
- Gossip protocal encryption (SERF), uses a symmetric (shared key) across all consul clients and servers
- Built-in ACL system
- Consul agent communication: TLS to encrypt RPC and API communication
- mTLS for service mesh - authenticity (talking to correct service) and encryption
- certificate authority - Consul own CA, or Vault, or external CA

- Gossip Protocol (serf) - encrypt communications throughout cluster
    - Uses a symmetric key
    - Shared secret for servers and clients in cluster
    - More in objective 9
- ACL - protects data and APIs
    - Not enabled by default
    - Protects access to Consul data and APIs
    - More on ACL system in objective 8
- Consul Agent - supports encrypting all communications using TLS (RPC/API)
    - Supports TLS communication to encrypt communications for RPC and API
    - Allows consul to be run over untrusted networks (public cloud, internet etc..)
    - Enabled in server configuration file
    - Consul can verify incoming/outgoing communications and check server hostnames
- mTLS to validate authenticity and encrypt communications
    - Uses the CA to validate authenticity against public CA bundle
    - Used for service mesh functionality
- CA - natively or integrate with an existing CA (Vault or Other)
    - Consul can acs as CA to issue certificates for the datacenter
    - Certificate types include:
        - Server - `consul tls cert create -server`
        - Client - `consul tls cert create -client`
        - CLI - `consul tls cert create -cli`

### Objective 7b: Differennt ceritificate types needed for TLS encryption
- Consul HTTP API and RPC require TLS certs
    - Encryption
- Service mesh connectivity uses mTLS certificates
    - Authenticty
    - Encryption
- You can manually set the HTTP port on Consul -https-port and provide a certificate if you want to manually configure the API to use HTTPS
- Consul can acs as CA
    - Enabled if connect is enabled
    - Consul can automatically distribute client certificates (automated)
    - Or you can do it manually with the operato method
- Certificates can be generated from your own CA
    - You distribute certs manually, also known as operator method
- Certificates must be signed by the same CA
- You can update to a new provider (migrate) at any time.

### Objective 7c: Understand the different TLS encryption settings for a fully secure datacenter
- Three primary configurations when working with Consul TLS.
    - `verify_server_hostname`:
        - All outgoing connections perform hostname verification
        - Ensures that servers have a certificate valid for `server.<datacenter>.<domain>`
        - Ensures a client cannot modify the consul agent config and restart as a server
        - Without this setting, consul onvly verifies that the cert is signed by a trusted CA
    - Cloud CA server certificates will include this hostname by default
    - If you are using yourt own CA, you MUST include`server.<datacenter>.<domain>` as a SAN
    - `verify_incoming`:
        - Requires that all incoming connections use TLS
        - The TLS certificate must be signed by a CA included in the ca_file or ca_path
        - By default setting is false
        - Setting is valid for both RPC and HTTP
    - `verify_outgoing`
        - Requires that all outgoing connections use TLS
        - TLS cert must be signed by a CA included in ca_file or ca_path
        - Default by false
        - Applies to both clients and servers
- Set in consul agent config file.