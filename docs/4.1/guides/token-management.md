## Manage Tokens

TODO: WIP

This guide will show you how to add, remove, and view active tokens.

There are several kinds of tokens that you will see as an admin:

* Node Join Tokens
* User Join Tokens
* Trusted Cluster Tokens

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