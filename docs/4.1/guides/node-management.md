# Managing Nodes

[TOC]

## Adding Nodes to the Cluster

## Step 1: Generate or Set a Join Token

There are two ways to invite nodes to join a cluster:

* Dynamic Tokens: Most secure
* Static Tokens: Less secure

Read more about these different kinds of tokens in the [Node Guide](../concepts/nodes/#joining-a-cluster)

### Option 1 (Recommended): Generate a dynamic token

You can generate or set a short-lived token with the [`tctl`](../cli-docs/#tctl) admin tool.

We recommend this method rather than Static Tokens because dynamic tokens automatically expire, preventing potentially malicious actors from adding nodes to your cluster.

The [`tctl nodes add`](../cli-docs/#tctl-nodes-add) command also shows the current CA Pin, which validates the current private key of the Auth Server before allowing a node to join the cluster. Read more about CA Pinning in the [Auth Server Guide](../concepts/auth/#ca-pinning).

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

### Option 2: Set a static token in config

You can set a static token in the `auth_service` section of your configuration file. The list of `tokens` set here are key-value pairs which represent the role(s) of a cluster node and tokens that they can use. We don't recommend this approach for security reasons, but it may be the best solution for some teams with organizational or networking constraints.

The tokens set in the config will not expire unless they are removed from the config and the `teleport` daemon is restarted. Anyone with the token and access to the auth server's network can add nodes to the cluster.

If you are adding a node using a static token we recommend that you add the `ca_pin` key to the `teleport` section on the node to be added. We show an example of how this looks below.

```bash
# get the CA Pin on the Auth server
$ tctl status
Cluster  grav-00
User CA  never updated
Host CA  never updated
CA pin   sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d
```

```yaml
# Config on the auth server
auth_service:
    enabled: true
    tokens:
    # This static token allows new hosts to join the cluster as
    # "node" role with the token `secret-token-value`.
    - "node:secret-token-value"
---
# Config on the node to be added
teleport:
  ca_pin: sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d
```
You will need to restart teleport for this configuration to take effect.
In production we recommend using a [Systemd Unit File](./production) to manage the `teleport` daemon.

```bash
$ systemctl reload teleport
# Tail the teleport service logs to confirm that it worked
$ journalctl -fu teleport
```

## Step 2: Use a Node Join Token

In the previous step [`tctl nodes add`](../cli-docs/#tctl-nodes-add) printed out a `teleport start` command which you can run on the node you want to add. You can use this command for testing, but be cautious! If the process fails unexpectedly or the node restarts the node service will not restart automatically.

On nodes that will remain in the cluster for a long time we recommend that you create a configuration file at `/etc/teleport.yaml` on the node and run teleport with as a [system unit](../production).

### Option 1 (Recommended): Run the node service with a config file

```yaml
teleport:
  auth_token: "fuzzywuzzywasabear"
  auth_servers:
    - 10.164.0.7:3025
ssh_service:
  enabled: "yes"
auth_service:
  enabled: "no"
proxy_service:
  enabled: "no"
```

### Option 2: Start the node service via the CLI

```bash
# adding a new regular SSH node to the cluster:
$ teleport start --roles=node --token=secret-token-value --auth-server=10.0.10.5

# adding a new regular SSH node using Teleport Node Tunneling:
$ teleport start --roles=node --token=secret-token-value --auth-server=teleport-proxy.example.com:3080

# adding a new proxy service on the cluster:
$ teleport start --roles=proxy --token=secret-token-value --auth-server=10.0.10.5
```

As new nodes come online, they start sending ping requests every few seconds
to the CA of the cluster. This allows users to explore cluster membership
and size:

```bsh
$ tctl nodes ls

Node Name     Node ID                                  Address            Labels
---------     -------                                  -------            ------
turing        d52527f9-b260-41d0-bb5a-e23b0cfe0f8f     10.1.0.5:3022      distro:ubuntu
dijkstra      c9s93fd9-3333-91d3-9999-c9s93fd98f43     10.1.0.6:3022      distro:debian
```

### Untrusted Auth Servers

Teleport nodes use the HTTPS protocol to offer the join tokens to the auth
server running on `10.0.10.5` in the example above. In a zero-trust
environment, you must assume that an attacker can highjack the IP address of
the auth server e.g. `10.0.10.5`.

To prevent this from happening, you need to supply every new node with an
additional bit of information about the auth server. This technique is called
"CA Pinning". It works by asking the auth server to produce a "CA Pin", which
is a hashed value of it's private key, i.e. it cannot be forged by an attacker.

On the auth server:

```bash
$ tctl status
Cluster  staging.example.com
User CA  never updated
Host CA  never updated
CA pin   sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1
```

The "CA pin" at the bottom needs to be passed to the new nodes when they're starting
for the first time, i.e. when they join a cluster:

Via CLI:

```bash
$ teleport start \
   --roles=node \
   --token=1ac590d36493acdaa2387bc1c492db1a \
   --ca-pin=sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1 \
   --auth-server=10.12.0.6:3025
```

or via `/etc/teleport.yaml` on a node:

```yaml
teleport:
  auth_token: "1ac590d36493acdaa2387bc1c492db1a"
  ca_pin: "sha256:7e12c17c20d9cb504bbcb3f0236be3f446861f1396dcbb44425fe28ec1c108f1"
  auth_servers:
    - "10.12.0.6:3025"
```

!!! warning "Warning":
    If a CA pin not provided, Teleport node will join a cluster but it will print
    a `WARN` message (warning) into it's standard error output.

!!! warning "Warning":
    The CA pin becomes invalid if a Teleport administrator performs the CA
    rotation by executing `tctl auth rotate`.

## Revoking Invitations

As you have seen above, Teleport uses tokens to invite users to a cluster (sign-up tokens) or
to add new nodes to it (provisioning tokens).

Both types of tokens can be revoked before they can be used. To see a list of outstanding tokens,
run this command:

```bsh
$ tctl tokens ls

Token                                Role       Expiry Time (UTC)
-----                                ----       -----------------
eoKoh0caiw6weoGupahgh6Wuo7jaTee2     Proxy      never
696c0471453e75882ff70a761c1a8bfa     Node       17 May 16 03:51 UTC
6fc5545ab78c2ea978caabef9dbd08a5     Signup     17 May 16 04:24 UTC
```

In this example, the first token has a "never" expiry date because it is a static token configured via a config file.

The 2nd token with "Node" role was generated to invite a new node to this cluster. And the
3rd token was generated to invite a new user.

The latter two tokens can be deleted (revoked) via `tctl tokens del` command:

```yaml
$ tctl tokens del 696c0471453e75882ff70a761c1a8bfa
Token 696c0471453e75882ff70a761c1a8bfa has been deleted
```

## Labeling Nodes

In addition to specifying a custom nodename, Teleport also allows for the
application of arbitrary key:value pairs to each node, called labels. There are
two kinds of labels:

1. `static labels` do not change over time, while `teleport` process is
   running. Examples of static labels are physical location of nodes, name of
   the environment (staging vs production), etc.

2. `dynamic labels` also known as "label commands" allow to generate labels at runtime.
   Teleport will execute an external command on a node at a configurable frequency and
   the output of a command becomes the label value. Examples include reporting load
   averages, presence of a process, time after last reboot, etc.

There are two ways to configure node labels.

1. Via command line, by using `--labels` flag to `teleport start` command.
2. Using `/etc/teleport.yaml` configuration file on the nodes.


To define labels as command line arguments, use `--labels` flag like shown below.
This method works well for static labels or simple commands:

```yaml
$ teleport start --labels uptime=[1m:"uptime -p"],kernel=[1h:"uname -r"]
```

Alternatively, you can update `labels` via a configuration file:

```yaml
ssh_service:
  enabled: "yes"
  # Static labels are simple key/value pairs:
  labels:
    environment: test
```

To configure dynamic labels via a configuration file, define a `commands` array
as shown below:

```yaml
ssh_service:
  enabled: "yes"
  # Dynamic labels AKA "commands":
  commands:
  - name: arch
    command: ['/path/to/executable', 'flag1', 'flag2']
    # this setting tells teleport to execute the command above
    # once an hour. this value cannot be less than one minute.
    period: 1h0m0s
```

`/path/to/executable` must be a valid executable command (i.e. executable bit must be set)
which also includes shell scripts with a proper [shebang line](https://en.wikipedia.org/wiki/Shebang_(Unix)).

**Important:** notice that `command` setting is an array where the first element is
a valid executable and each subsequent element is an argument, i.e:

```yaml
# valid syntax:
command: ["/bin/uname", "-m"]

# INVALID syntax:
command: ["/bin/uname -m"]

# if you want to pipe several bash commands together, here's how to do it:
# notice how ' and " are interchangeable and you can use it for quoting:
command: ["/bin/sh", "-c", "uname -a | egrep -o '[0-9]+\.[0-9]+\.[0-9]+'"]
```


When a user executes [`tsh --proxy=p ssh saturn`](../cli-docs/#tsh-ssh) command, trying to log into the Node "saturn", the [`tsh`](../cli-docs/#tsh) tool will establish HTTPS connection to the proxy "p" and authenticate before it will be given access to "saturn".