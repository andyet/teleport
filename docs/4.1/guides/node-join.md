## Adding Nodes to the Cluster

This guide will show you a few different ways to generate and use join tokens.
Join tokens are used by nodes to prove that they are trusted by an admin and
should be allowed to join a cluster. Once a node has joined a cluster it can
see the IP addresses and labels of other nodes along with Teleport User data.

[TOC]

## Recommended Prerequisites

* Read through the [Architecture Overview](../architecture/overview).
* Read through the [Production Guide](./production) if you are setting up
Teleport in production.
* Run _all_ nodes as [Systemd Units](./production/#systemd-unit-file) unless you
are just working in a sandbox.

## Step 1: Generate or Set a Join Token

There are two ways to invite nodes to join a cluster:

* **Dynamic Tokens**: Most secure
* **Static Tokens**: Less secure

### Option 1 (Recommended): Generate a dynamic token

You can generate or set a short-lived token with the
[`tctl`](../cli-docs/#tctl) admin tool.

We recommend this method rather than static tokens because dynamic tokens
automatically expire, preventing potentially malicious actors from adding nodes
to your cluster.

The [`tctl nodes add`](../cli-docs/#tctl-nodes-add) command also shows the
current CA Pin, which validates the current private key of the Auth Server
before allowing a node to join the cluster. Read more about CA Pinning in the
[Production Guide](./production/#ca-pinning).

```bsh
# Set a specific token value that must be used within 5 minutes
$ tctl nodes add --ttl=5m --roles=node --token=secret-token-value
The invite token: secret-token-value
This token will expire in 5 minutes

Run this on the new node to join the cluster:

> teleport start \
   --roles=node \
   --token=secret-token-value \
   --ca-pin=sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d \
   --auth-server=10.164.0.7:3025

Please note:

  - This invitation token will expire in 5 minutes
  - 10.164.0.7:3025 must be reachable from the new node
```

If `--token` is not provided `tctl` will generate one.
```bsh
# generate a dynamic invitation token for a new node:
$ tctl nodes add --ttl=5m --roles=node
The invite token: e94d68a8a1e5821dbd79d03a960644f0
This token will expire in 5 minutes

Run this on the new node to join the cluster:

> teleport start \
   --roles=node \
   --token=e94d68a8a1e5821dbd79d03a960644f0 \
   --ca-pin=sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d \
   --auth-server=10.164.0.7:3025

Please note:

  - This invitation token will expire in 5 minutes
  - 10.164.0.7:3025 must be reachable from the new node
```

The command prints out a `teleport start` command which you can run on a node
that you want to add to a cluster. We only recommend this option for sandbox or
staging environments as you are getting started with teleport.

!!! warning "Resiliency Warning"
    If the process fails unexpectedly or the node restarts the node service will not
    restart automatically. See how to [add a node with a config
    file](#option-1-recommended-run-the-node-service-with-a-config-file) for a more
    resilient method.

### Option 2: Set a static token in config file

You can set a static token in the `auth_service` section of your configuration
file. The list of `tokens` represent the role(s) of a cluster node and tokens
that they can use. We encourage the use of [dynamic tokens](#option-1-recommended-generate-a-dynamic-token) for security,
but using static token may be the best option for some teams.

The tokens set in the config will not expire unless they are removed from the
config and the `teleport` daemon is restarted. Anyone with the token and access
to the auth server's network can add nodes to the cluster.

If you are adding a node using a static token we recommend that you add the
[`ca_pin`](./production/#ca-pinning) key to the `teleport` section on the node
to be added. Here's an example of how this works.

```bash
# get the CA Pin on the Auth server
$ tctl status
Cluster  grav-00
User CA  never updated
Host CA  never updated
CA pin   sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d
```

Edit the yaml config on the Auth Server.

```diff
# Add static toke "secret-token-value"
# which will allow nodes with role `node` to join
auth_service:
    enabled: true
    tokens:
    # This static token allows new hosts to join the cluster as
    # "node" role with the token `secret-token-value`.
+    - "node:secret-token-value"
```

You will need to restart teleport for the static token configuration to take
effect. In production we recommend using a [Systemd Unit File](./production) to
manage the `teleport` daemon so we show `systemctl` commands below. If you
are not using `systemctl` currently just `Ctrl-C` to or `kill <pid>` to
kill the current teleport process.

```bash
$ systemctl reload teleport
# Tail the teleport service logs to confirm that it worked
$ journalctl -fu teleport
```

<!-- TODO Add Node Behind NAT -->

## Step 2: Use a Node Join Token

In the previous step [`tctl nodes add`](../cli-docs/#tctl-nodes-add) printed out
a `teleport start` command which you can run on the node you want to add. You
can use this command for testing, but be cautious! If the process fails
unexpectedly or the node restarts the node service will not restart
automatically. Fix this by running [Teleport as a System
Unit](./production/#systemd-unit-file).

### Option 1 (Recommended): Run the node service with a config file

Check that the join token you are using is recognized by the Auth Service
```bash
# run this on an auth node
# here we have a static which will never expire
# and a dynamic token which will expire
$ tctl tokens ls
Token                            Type Expiry Time (UTC)
-------------------------------- ---- -------------------
fuzzywuzzywasabear               Node never
b150b9349b4ca40bcb4df298a2f50152 Node 17 Oct 19 11:18 UTC
```

Add the token and [CA Pin](../production/#ca-pinning) to your config file

```diff
# Node Service Config
# You may have other config
# this example shows a minimal config
# for running only the `node` service
teleport:
+  auth_token: "fuzzywuzzywasabear"
+  ca_pin: "sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d"
  auth_servers:
    - 10.164.0.7:3025
ssh_service:
  enabled: "yes"
auth_service:
  enabled: "no"
proxy_service:
  enabled: "no"
```

Save these files and restart/start both services

```bash
# on auth server
$ systemctl reload teleport
```

```bash
# on node which is joining
$ systemctl start teleport
```

Use `systemctl status teleport` or `journalctl -u teleport` to check the logs
and make sure there are no errors.

!!! warning "Certificate Warnings"
    If you previously joined the cluster using an old token or certificate
    you may see an error `x509: certificate signed by unknown authority`.
    This is due to a mismatch between the Auth Server state and the information presented by the node. To resolve it you can remove the node certificate
    with `rm -r /var/lib/teleport` on the node and/or
    `tctl rm nodes/<node-uuid>` on the auth node to make Teleport Auth "forget"
    the node and start fresh.

### Option 2: Start the node service via the CLI

This option can be used when to quickly add a node to a cluster with minimal
configuration. We only recommend this option for sandbox or staging environments
as you are getting started with teleport. Get the CA Pin of the auth node by
running `tctl status`.

```bash
# adding a new regular SSH node to the cluster:
$ teleport start --roles=node --token=b150b9349b4ca40bcb4df298a2f50152
> --auth-server=10.164.0.7:3025
> --ca-pin=sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d
```

## Next Steps

As new nodes come online, they start sending ping requests every few seconds
to the Auth service. You can see the nodes that have successfully joined
by running [`tctl nodes ls`](../cli-docs/#tctl-nodes-ls)

```bsh
$ tctl nodes ls

Node Name     Node ID                                  Address            Labels
---------     -------                                  -------            ------
turing        d52527f9-b260-41d0-bb5a-e23b0cfe0f8f     10.1.0.5:3022      distro:ubuntu
dijkstra      c9s93fd9-3333-91d3-9999-c9s93fd98f43     10.1.0.6:3022      distro:debian
```

!!! tip "Join Tokens are only used for the initial connection"
    It is important to understand that join tokens are only used to establish
    the connection for the first time. The clusters will exchange certificates
    and won't be using the token to re-establish the connection in the future.
    Future connections will rely upon the [node certificate](../architecture/auth/#authentication-in-teleport) to identify and authorize a node.

**More Guides**
* Add Labels to Nodes
* Revoke Tokens
