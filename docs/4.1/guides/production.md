# Production Guide

TODO Build off Quickstart, but include many more details and multi-node set up.

Minimal Config example

Include security considerations. Address vulns in quay?

[TOC]

## Prerequisites

* Read about [Teleport Basics](../concepts/basics)
* Read through the [Installation Guide](../guides/installation) to see the available packages and binaries available.
* Read the CLI Docs for [`teleport`](../cli-docs/#teleport)

## Designing Your Cluster

Before installing anything there are a few things you should think about.

* Where will you host Teleport
    * On-premises
    * Cloud VMs such as AWS EC2 or GCE
    * An existing Kubernetes Cluster
* What does your existing network configuration look like?
    * Are you able to administer the network firewall rules yourself or do you need to work with a network admin?
    * Are these nodes accessible to the public Internet or behind NAT?
* Which users ([Roles or ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) on k8s) are set up on the existing system?
   * Can you add new users or Roles yourself or do you need to work with a system admin?

## Firewall Configuration

Teleport services listen on several ports. This table shows the default port numbers.

|Port      | Service    | Description | Ingress | Egress
|----------|------------|-------------|---------|----------
| 3080      | Proxy      | HTTPS port clients connect to. Used to authenticate `tsh` users and web users into the cluster. | Allow inbound connections from HTTP and SSH clients.| Allow outbound connections to HTTP and SSH clients.
| 3023      | Proxy      | SSH port clients connect to after authentication. A proxy will forward this connection to port `3022` on the destination node. | Allow inbound traffic from SSH clients. | Allow outbound traffic to SSH clients.
| 3022      | Node       | SSH port to the Node Service. This is Teleport's equivalent of port `22` for SSH. | Allow inbound traffic from proxy host. | Allow outbound traffic to the proxy host.
| 3025      | Auth       | SSH port used by the Auth Service to serve its Auth API to other nodes in a cluster. | Allow inbound connections from all cluster nodes. | Allow outbound traffic to cluster nodes.
| 3024      | Proxy      | SSH port used to create "reverse SSH tunnels" from behind-firewall environments into a trusted proxy server. | <TODO> | <TODO>

<TODO: Add several diagrams of firewall config examples>


## Installation

First

## Running Teleport in Production

### Systemd Unit File

In production, we recommend starting teleport daemon via an init system like
`systemd`. If systemd and unit files are new to you check out [this helpful guide](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files). Here's the recommended Teleport service unit file for systemd.


```yaml
[Unit]
Description=Teleport SSH Service
After=network.target

[Service]
Type=simple
Restart=on-failure
# Set the nodes roles with the `--roles`
# In most production environments you will not
# want to run all three roles on a single host
# proxy,auth,node is the default value if none is set
ExecStart=/usr/local/bin/teleport start --roles=auth --config=/etc/teleport.yaml --pid-file=/var/run/teleport.pid
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/var/run/teleport.pid

[Install]
WantedBy=multi-user.target
```

There are a couple of important things to notice about this file:

1. The start command in the unit file specifies `--config` as a file and there are very few flags passed to the `teleport` binary. Most of the configuration for Teleport should be done in the [configuration file](../configuration).

2. The **ExecReload** command allows admins to run `systemctl reload teleport`. This will attempt to perform a graceful restart of _*but it only works if network-based backend storage like [DynamoDB](../configuration/#storage) or [etc 3.3](../configuration/#storage) is configured*_. Graceful Restarts will fork a new process to handle new incoming requests and leave the old daemon process running until existing clients disconnect.

You can also perform restarts/upgrades by sending `kill` signals
to a Teleport daemon manually.

| Signal                  | Teleport Daemon Behavior
|-------------------------|---------------------------------------
| `USR1`                  | Dumps diagnostics/debugging information into syslog.
| `TERM`, `INT` or `KILL` | Immediate non-graceful shutdown. All existing connections will be dropped.
| `USR2`                  | Forks a new Teleport daemon to serve new connections.
| `HUP`                   | Forks a new Teleport daemon to serve new connections **and** initiates the graceful shutdown of the existing process when there are no more clients connected to it.



This will copy Teleport binaries to `/usr/local/bin`.

Let's start Teleport. First, create a directory for Teleport
to keep its data. By default it's `/var/lib/teleport`. Then start `teleport` daemon:

```bash
$ sudo teleport start
```

!!! danger "WARNING":
    Teleport stores data in `/var/lib/teleport`. Make sure that regular/non-admin users do not
    have access to this folder on the Auth server.


If you are logged in as `root` you may want to create a new OS-level user first. On linux create a new user called `<username>` with the following commands:
```bash
$ adduser <username>
$ su <username>
```

Security considerations on installing tctl under root or not

!!! danger "WARNING":
    Teleport stores data in `/var/lib/teleport`. Make sure that regular/non-admin users do not
    have access to this folder on the Auth server.-->
