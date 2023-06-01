# Validators

Validators run the servers that allow users to download and create blocks. They validate,
execute and cryptographically certify the blocks of all the chains.

> In Linera, every chain is backed by the same set of validators and has the same level of
> security.

The main function of validators is to guarantee the integrity of the infrastructure in the sense that:

- Each block is valid, i.e. it has the correct format, its operations are allowed, the
  received effects are in the correct order, and e.g. the balance was correctly computed.

- Every effect received by one chain was actually sent by another chain.

- If one block on a particular height is certified, no other block on the same height is.

These properties are guaranteed to hold as long as two third of the validators (weighted
by their stake) follow the protocol. In the future, deviating from the protocol may cause
a validator to be considered malicious and to lose their _stake_.

Validators also play a role in the liveness of the system by making sure that the history
of the chains stays available. However, since validators do not propose blocks on most
chains (see [next section](block_creation.html)), they do _not_ guarantee that any
particular operation or effect will eventually be executed on a chain. Instead, chain
owners decide whether and when to propose new blocks, and which operations and effects to
include. The current implementation of the Linera client automatically includes all
incoming effects in new blocks. The operations are the actions the chain owner explicitly
adds, e.g. transfer.

## Scalability

Since every chain uses the same validators, adding more chains does not require adding
validators. Instead, it requires each individual validator to scale up and handle the
additional load.

That is why Linera allows the same validator to run multiple `server`
processes—_workers_—on different machines, each one processing a different subset of
the microchains. The workers communicate directly with each other whenever a chain sends
an effect to another chain.

## Anatomy of a Validator

To achieve scalability, Linera validators need to be horizontally scalable (sometimes
referred to as "elastic"). To achieve this scalability, the architecture of a validator
involves splitting the chains up over multiple "shards" or "workers" which are
encapsulated by a single ingress/egress called the "proxy".

The validator has an internal network enabling the proxy to speak with shards and shards
with each other. Each shard is also backed by its own data store which can be scaled
independently of the rest of the validator

```ignore
 example network
                     │                                           │
                     │                                           │
                     │                                           │
┌────────────────────┼────────────────────┐ ┌────────────────────┼────────────────────┐
│ validator 1        │                    │ │ validator 2        │                    │
│              ┌─────┴─────┐              │ │              ┌─────┴─────┐              │
│              │   proxy   │              │ │              │   proxy   │              │
│        ┌─────┤           ├─────┐        │ │        ┌─────┤           ├─────┐        │
│        │     └───────────┘     │        │ │        │     └───────────┘     │        │
│        │                       │        │ │        │                       │        │
│        │                       │        │ │        │                       │        │
│  ┌─────┴─────┐           ┌─────┴─────┐  │ │  ┌─────┴─────┐           ┌─────┴─────┐  │
│  │   shard   │           │   shard   │  │ │  │   shard   │           │   shard   │  │
│  │     1     │           │     2     │  │ │  │     1     │           │     2     │  │
│  └─────┬─────┘           └─────┬─────┘  │ │  └─────┬─────┘           └─────┬─────┘  │
│        │                       │        │ │        │                       │        │
│  ┌─────┴─────┐           ┌─────┴─────┐  │ │  ┌─────┴─────┐           ┌─────┴─────┐  │
│  │    db1    │           │    db2    │  │ │  │    db1    │           │    db2    │  │
│  │           │           │           │  │ │  │           │           │           │  │
│  └───────────┘           └───────────┘  │ │  └───────────┘           └───────────┘  │
│                                         │ │                                         │
└─────────────────────────────────────────┘ └─────────────────────────────────────────┘

```

## Configuring Networks, Workers, and Proxies

In [a previous section](../getting_started/first_app.md), we used the `run_local.sh` script
to start a local network. This should be sufficient for most use cases when you're running
a local network.

```bash
./scripts/run_local.sh
```

However, it is possible to customize and configure the parameters of the network.

`run_local.sh` uses the `validator_n.toml` file from the `configuration/` directory to configure validator number `n`.

```bash
server generate --validators configuration/validator_{1,2,3,4}.toml --committee committee.json
```

generates keys and writes them, together with the options from the TOML files, to
`server_1.json`, ..., `server_4.json`. It also stores the set of the new validators'
public keys in `committee.json`.

```bash
linera --wallet wallet.json create-genesis-config 10 --genesis genesis.json --initial-funding 10 --committee committee.json
```

creates a configuration for the initial state of the network, `genesis.json`, with 10
chains, each with a balance of 10. It also creates a `wallet.json` for a client who owns
all those chains.

To start the newly configured network, each validator `n` must start their proxy:

```bash
proxy server_n.json &
```

And all shards; for shard `i`:

```bash
server run --storage rocksdb:server_n_i.db --server server_n.json --shard i --genesis genesis.json &
```

This will create a separate database file `server_n_i.db` for each shard. In a production
network, these would be running on different machines.

## Changing the Set of Validators

If a new validator wants to start participating, or an old one wants to leave, all chains
must be updated.

The system has one designated _admin chain_, where the validators can join or leave or,
and where new _epochs_ are defined. During every epoch, the set of validators is fixed. If
you own the admin chain, you can use the `set-validator` and `remove-validator` commands
to start a new epoch with a modified set of validators:

```bash
linera --wallet wallet.json set-validator --name 5b611b86cc1f54f73a4abfb4a2167c7327cc85a74cb2a5502431f67b554850b4 --address 127.0.0.1:9100 --votes 3
linera --wallet wallet.json remove-validator --name f65a585f05852f0610e2460a99c23faa3969f3cfce8a519f843a793dbfb4cb84
```

Chain owners must then create a block that receives the `SetCommittees` effect from the
admin chain, and have it certified by the old validators. Only the _next_ block in their
chain will be certified by the new validator set!

The _admin chain_ is currently managed by a single user. In the future, it will be a
_public chain_ (i.e. managed by validators). We anticipate that Linera epochs will change
once per day (or less) and that several subsequent epochs will overlap so that chain
owners have enough time to migrate their chains. (Chain migration may also be delegated to
third parties. See [next section](block_creation.html).)