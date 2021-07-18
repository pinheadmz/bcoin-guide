# Week 2: Nodes and P2P Connections

## Connect two full nodes in regtest

This will require two terminal windows but might be easier to manage using `tmux`
with two panes on the same screen. We are going to connect multiple nodes
and watch them exchange messages.

You may want to clear out your regtest chain first with `rm -rf ~/.bcoin/regtest`

Terminal 1: Start bcoin in regtest with default options.

```
bcoin --network=regtest
```

Terminal 2: Start a second bcoin full node:

```
bcoin --network=regtest
```

You should get an error here, either that the data directory is already in use
or that the http / p2p ports are in use (by the first node). Two run the second
node in its own space, add these options:

```
bcoin \
  --network=regtest \
  --prefix=~/.bcoin/regtest-2 \
  --port=10000 \
  --http-port=20000 \
  --wallet-http-port=30000
```

Reminder: you can learn about these options and more [here](https://github.com/bcoin-org/bcoin/blob/master/docs/configuration.md).

Terminal 3: CLI commands

Using only default options (which will therefore target node #1, also set
with default options), generate 100 blocks to your default wallet address:

```
bcoin-cli rpc generatetoaddress 100 `bwallet-cli rpc getnewaddress`
```

Notice the second command embedded using backticks - do you understand what's happening?

Now check the chain height of both nodes:

```
bcoin-cli info | jq .chain
bcoin-cli --http-port=20000 info | jq .chain
```

The two nodes are out of sync! Let's connect them: https://bcoin.io/api-docs/?shell--cli#addnode

```
bcoin-cli --http-port=20000 rpc addnode 127.0.0.1 onetry
```

Now check both nodes' `info` again, they should be synced and you should have been
able to observe the second node syncing those 100 blocks from the first node.

## Demonstrate A Chain Reorganization

Disconnect the second node from the first: https://bcoin.io/api-docs/?shell--cli#disconnectnode

```
bcoin-cli --http-port=20000 rpc disconnectnode 127.0.0.1
```

Generate 2 blocks on the first node and 1 block on the second node:

```
bcoin-cli rpc generatetoaddress 2 `bwallet-cli rpc getnewaddress`
bcoin-cli --http-port=20000 rpc generatetoaddress 1 `bwallet-cli rpc getnewaddress`
```

If you check both nodes' `info` again, they should be out of sync. However, this time
what we have is a CHAIN SPLIT. Both nodes have the same blockchain up to height 100.
After that point, the second node has 1 new block but the first node has 2 totally different blocks.
The two chains after height 100 are completely different. What will happen when we
connect these two nodes back together again?

```
bcoin-cli --http-port=20000 rpc addnode 127.0.0.1 onetry
```

Once you execute this, observe the log from the second node. The actual log file
can be found at `~/.bcoin/regtest-2/debug.log` if you want to inspect it later but
the same output should be in the stdout of that terminal window / tmux pane.

You should see a few warnings like this:

```
[warning] (chain) Heads up: Competing chain at height 101: tip-height=101 competitor-height=101
  tip-hash=397c49bcd2159b7e3cd528ede1981aa8760b758ec2a141ba1c21de97226b307c competitor-hash=4c6f53e2d911aa1d3c2a8f437b37d7fbe39eafdf0df016a1bfdd1f81c5d25df7
  tip-chainwork=204 competitor-chainwork=204 chainwork-diff=0
[debug] (chain) Memory: rss=48mb, js-heap=10/12mb native-heap=36mb
[info] (chain) Block 4c6f53e2d911aa1d3c2a8f437b37d7fbe39eafdf0df016a1bfdd1f81c5d25df7 (101) added to chain (size=280 txs=1 time=6.685683).
[debug] (net) Received full compact block 0e463b7bf53efb2b2dc517196f37efaf58d09d5c8af636e0f99c4577fd538506 (127.0.0.1:48444).
[warning] (chain) WARNING: Reorganizing chain.
[debug] (wallet) Adding block: 101.
[warning] (chain) Chain reorganization:
  old=397c49bcd2159b7e3cd528ede1981aa8760b758ec2a141ba1c21de97226b307c(101)
  new=0e463b7bf53efb2b2dc517196f37efaf58d09d5c8af636e0f99c4577fd538506(102)
[debug] (wallet) Adding block: 102.
```

As an exercise, and to begin the deep-dive into bcoin development, review the
parts of the code that generate these warnings and try to follow along.
The relevant functions (in `lib/blockchain/chain.js`) are:
- [`connect()`](https://github.com/bcoin-org/bcoin/blob/950e30c084141e7c8cb81233136b9e5bc7e1e02c/lib/blockchain/chain.js#L1440-L1450)
- [`reorganize()`](https://github.com/bcoin-org/bcoin/blob/950e30c084141e7c8cb81233136b9e5bc7e1e02c/lib/blockchain/chain.js#L875-L887)
- [`setBestChain()`](https://github.com/bcoin-org/bcoin/blob/0551096c0a0dae4ad8fb9f1e135b743d50a19983/lib/blockchain/chain.js#L1043)
- [`saveAlternate()`](https://github.com/bcoin-org/bcoin/blob/0551096c0a0dae4ad8fb9f1e135b743d50a19983/lib/blockchain/chain.js#L1106)

## Double Spend Attack!

There are several types of [double-spend attacks](https://en.bitcoin.it/wiki/Irreversible_Transactions)
theoretically possible in Bitcoin. Since we have 100% of the mining hashrate in
our two-node regtest network, we can easily demonstrate how it works.

Given how we have executed our `generatetoaddress` commands thus far, the wallet
in the first node should have a big balance, and the second node's wallet should be empty:

```
$ bwallet-cli balance
{
  "account": -1,
  "tx": 102,
  "coin": 102,
  "unconfirmed": 510000000000,
  "confirmed": 510000000000
}

$ bwallet-cli --http-port=30000 balance
{
  "account": -1,
  "tx": 0,
  "coin": 0,
  "unconfirmed": 0,
  "confirmed": 0
}
```

We are going to send a transaction to the second node, and then double-spend-attack
that wallet, effectively reversing the transaction and potentially stealing from
that user!

Get an address from the victim's wallet:

```
bwallet-cli --http-port=30000 rpc getnewaddress
```

Send 150 BTC from the first node to that address:

```
bwallet-cli send <victim's address> 150.00000000 --subtract-fee=true
```

_Why 150 BTC?_ In short, it's to guarantee the success of this test.
You might be able to figure why we use this value on your own but here are some
hints you can search for in addition to learning exactly what a double-spend is:
- Coinbase maturity
- Coin selection

Confirm that TX in a block and note the block hash that is returned:

(example)
```
$ bcoin-cli rpc generatetoaddress 1 `bwallet-cli rpc getnewaddress`
[
  "4965f6fd776265eb23a31f2a3c85eaaefc3a49ac447e47bf6cda274e9f380e2a"
]
```

At this point the "victim" (the second node's wallet) appears to have a confirmed
balance of around 150 BTC (minus the transaction fee, do you know why?).
Here is where we are going to start getting nasty. Notice how
all the following commands are only executed to the first ("attacker") node, while remaining connected.
Some of these steps are a bit hacky to side-step safety features normally built in to
bcoin.

Undo the last block, un-confirming the 150 BTC transaction: https://bcoin.io/api-docs/#invalidateblock

```
bcoin-cli rpc invalidateblock <block hash from last command>
```

If you check the nodes' `info` right now, you'll notice that the first node
is now "behind" by one block -- the invalidation command is local only and does
not affect other nodes on the network.

Remove the now-unconfirmed transaction from the sender's wallet: https://bcoin.io/api-docs/?shell--cli#zap-transactions
Without this step, the wallet will not let us double-spend our coins.

```
bwallet-cli zap --account=default --age=1
```

Check that the wallet has no "pending" transactions, this would prevent us
from double-spending:

```
$ bwallet-cli pending
[]
```

OK, let's steal our money back:

```
bwallet-cli send `bwallet-cli rpc getnewaddress` 150.00000000 --subtract-fee=true
```

Now we confirm that transaction once:

```
bcoin-cli rpc generatetoaddress 1 `bwallet-cli rpc getnewaddress`
```

The second node should log the familiar "Heads up" warning.

Then we reorganize with one more block:

```
bcoin-cli rpc generatetoaddress 1 `bwallet-cli rpc getnewaddress`
```

WOW, the second node and its wallet really did not like that, huh?

```
[warning] (chain) WARNING: Reorganizing chain.
[debug] (wallet) Rescan: reverting block 103
[warning] (wallet) Disconnected wallet block 1103b6349e5c30a4e390132f2c3cfb108b52dfa8a1b87a1a2790df066027c458 (tx=1).
[debug] (wallet) Adding block: 103.
[info] (wallet) Incoming transaction for 1 wallets in WalletDB (2416818256d576c9ac8b24b7108ffb96b240e6d577a9745b49491b14f23ad929).
[warning] (wallet) Handling conflicting tx: ba3a87a2cc42327cf7d08bdb33ca8785d301f260dbc432aeb664c65ca241c8a9.
[warning] (wallet) Removed conflict: ba3a87a2cc42327cf7d08bdb33ca8785d301f260dbc432aeb664c65ca241c8a9.
[warning] (chain) Chain reorganization:
  old=1103b6349e5c30a4e390132f2c3cfb108b52dfa8a1b87a1a2790df066027c458(103)
  new=62cab7c55d967b45b765fefabfe5aa99287c0f7c0161bec8c8a0ae33eadd3eab(104)
[debug] (wallet) Adding block: 104.
```

Now check the balances of the two nodes:

```
$ bwallet-cli balance
{
  "account": -1,
  "tx": 203,
  "coin": 103,
  "unconfirmed": 877500000000,
  "confirmed": 877500000000
}

$ bwallet-cli --http-port=30000 balance
{
  "account": -1,
  "tx": 0,
  "coin": 0,
  "unconfirmed": 0,
  "confirmed": 0
}
```

# BUMMER.

## Homework

Remember the second node in our regtest network can ALSO generate blocks.
Can you STEAL BACK the 150 BTC that was originally sent to the second node?

It will involve getting the raw transaction hex (is it still printed to your
terminal somewhere?) and doing our chain reorganization trick again, but this
time broadcasting from the receiver's node using: https://bcoin.io/api-docs/?shell--cli#broadcast-transaction
