## Label Nodes

[TOC]

Teleport allows for the application of arbitrary key-value pairs to each node,
called labels. There are two kinds of labels:

1. `static labels` do not change over time, while `teleport` process is running.
   Examples of static labels are physical location of nodes, name of the
   environment (staging vs production), etc.

2. `dynamic labels` also known as "label commands" allow to generate labels at
   runtime. Teleport will execute an external command on a node at a
   configurable frequency and the output of a command becomes the label value.
   Examples include reporting load averages, presence of a process, time after
   last reboot, etc.

There are two ways to configure node labels.

1. Via command line, by using `--labels` flag to `teleport start` command.
2. Using `/etc/teleport.yaml` configuration file on the nodes.

## Example 1: Add labels on the command line

To define labels as command line arguments, use `--labels` flag like shown below
when you start the `teleport --roles=node` service. This method works well for
static labels or simple commands.

In this example the node will have the static label `env=sanbox`. Teleport will
run `uptime -p` every minute and assign the result to the label key `uptime`. It
will also run `uname -r` and assign it to the label key `kernel`

```bash
# first kill previous instances of teleport
# with Ctrl-C or kill <pid>
$ teleport start --roles=node
> --labels env=sandbox,uptime=[1m:"uptime -p"],kernel=[1h:"uname -r"]
> --token=secret-token-value \
> --ca-pin=sha256:1146cdd2b887772dcc2e879232c8f60012a839f7958724ce5744005474b15b9d \
> --auth-server=10.164.0.7:3025
```

!!! warning "Resiliency Warning"
    If the process fails unexpectedly or the node restarts the node service will not
    restart automatically. Run the `teleport` binary as a [Systemd Unit](./production/#systemd-unit-file) to avoid this.

## Example 2: Add labels to the config file

Alternatively, you can update `labels` via a configuration file:

```yaml
ssh_service:
  enabled: "yes"
  # Static labels are simple key/value pairs:
  labels:
    environment: test
```

To configure dynamic labels via a configuration file, define a `commands` array
as shown below. The `name` key is the label key: `arch` in the example here.

```yaml
ssh_service:
  enabled: "yes"
  # Dynamic labels are listed under "commands":
  commands:
  - name: arch
    command: ['/usr/bin/uname', '-r', '-s']
    # this setting tells teleport to execute the command uname
    # once an hour. `period` cannot be less than one minute.
    period: 1h0m0s
```

`/path/to/executable` must be a valid executable command with the (i.e.
executable bit must be set). If you run the `teleport` daemon as `root` this
should not be an issue, but if `teleport` runs as a non-root user in your system
check the permissions of the executable with `ls -l <executable-filepath>`.
Modify file permissions from an authorized OS user with the `chmod +x` command.
If the executable is a shell script it must have a proper [shebang
line](https://en.wikipedia.org/wiki/Shebang_(Unix)).

**Syntax Tip:** notice that `command` setting is an array where the first element
is a valid executable and each subsequent element is an argument, i.e:

```yaml
# valid syntax:
command: ["/bin/uname", "-m"]

# INVALID syntax:
command: ["/bin/uname -m"]

# if you want to pipe several bash commands together, here's how to do it:
# notice how ' and " are interchangeable and you can use it for quoting:
command: ["/bin/sh", "-c", "uname -a | egrep -o '[0-9]+\.[0-9]+\.[0-9]+'"]
```
**More Guides**
* Add Nodes to a Cluster
* Revoke Tokens
