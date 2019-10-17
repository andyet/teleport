### Untrusted Auth Servers

Teleport nodes use the HTTPS protocol to offer the join tokens to the auth
server running on `10.0.10.5` in the example above. In a zero-trust environment,
you must assume that an attacker can highjack the IP address of the auth server
e.g. `10.0.10.5` .

To prevent this from happening, you need to supply every new node with an
additional bit of information about the auth server. This technique is called
"CA Pinning". It works by asking the auth server to produce a "CA Pin", which is
a hashed value of it's private key, i.e.it cannot be forged by an attacker.

On the auth server:

``` bash
$ tctl status
Cluster  staging.example.com
User CA  never updated
Host CA  never updated
CA pin   sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1
```

The "CA pin" at the bottom needs to be passed to the new nodes when they're
starting for the first time, i.e.when they join a cluster:

Via CLI:

``` bash
$ teleport start \
   --roles=node \
   --token=1ac590d36493acdaa2387bc1c492db1a \
   --ca-pin=sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1 \
   --auth-server=10.12.0.6:3025
```

or via `/etc/teleport.yaml` on a node:

``` yaml
teleport:
  auth_token: "1ac590d36493acdaa2387bc1c492db1a"
  ca_pin: "sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1"
  auth_servers:

    - "10.12.0.6:3025"

```

!!! warning "Warning":
    If a CA pin not provided, Teleport node will join a
    cluster but it will print a `WARN` message (warning) into it's standard
    error output.

!!! warning "Warning":
    The CA pin becomes invalid if a Teleport administrator
    performs the CA rotation by executing
    [ `tctl auth rotate` ](../cli-docs/#tctl-auth-rotate) .