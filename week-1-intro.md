# Week 1: Intro to bcoin

## Introduction materials

SF Bitcoin Developers Presentation: https://www.youtube.com/watch?v=MGm54LZ1T50

SF NodeJS Meetup Presentation: https://www.youtube.com/watch?v=i9w7U4onn0M

Project Homepage with guides and API docs: https://bcoin.io/

Stack Exchange questions about bcoin: https://bitcoin.stackexchange.com/search?q=bcoin

## Clone and install bcoin

https://github.com/bcoin-org/bcoin/blob/master/docs/getting-started.md

```
git clone https://github.com/bcoin-org/bcoin
cd bcoin
npm rebuild
npm install --global
```

## Run the tests locally

From inside bcoin repo directory:

Run all tests:

```
npm run test
```

Run one test:

```
npm run test-file test/address-test.js
```

## Run bcoin in regtest mode

https://github.com/bcoin-org/bcoin/blob/master/docs/configuration.md

If you installed bcoin globally correctly, you should be able to execute this command "from anywhere":

```
bcoin --network=regtest
```

This will start a bcoin full node in a local test mode with no peers and 0-difficulty mining.
Remember, regtest mode is NOT REAL MONEY, so it's ok to make mistakes ;-)
THe bcoin full node will be launched with default options but by reading through the configuration guide above
you can see what other settings are available.

In a second terminal window (or tmux pane), use the bcoin-cli tool to get node info:

```
bcoin-cli --network=regtest info
```

You can also set an enviorment variable and then you don't need to pass `--network=...` to every command:

```
export BCOIN_NETWORK=regtest
bcoin-cli info
```

## Explore the API

https://bcoin.io/api-docs/

By default, bcoin runs a wallet and that wallet is initialized on first launch.
The wallet is called `primary` and has one account called `default`. For more
details about how the wallet is structured (like what do I mean by "account"?)
try reading through [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
and let me know if you have any questions about it.

### Bitcoin wallet practice

bcoin also uses [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
seed phrases for its wallets.

Get your wallet's seed phrase: https://bcoin.io/api-docs/?shell--cli#get-master-hd-key

```
bwallet-cli master
```

Write down that seed phrase and keep it safe! If this were mainnet, that phrase is
how you can backup and restore your wallet!

Get your wallet's current receive address: https://bcoin.io/api-docs/?shell--cli#generate-receiving-address

```
bwallet-cli --account=default address
```

Now that you can receive test Bitcoin we can mine some regtest blocks, generating
coins and funding your wallet: https://bcoin.io/api-docs/?shell--cli#generatetoaddress

```
bcoin-cli rpc generatetoaddress 110 <your address>
```

Did it work? https://bcoin.io/api-docs/?shell--cli#get-balance

```
bwallet-cli balance
```

Now let's create a second wallet, one that uses Segregated Witness. A brief explanation
of this wallet type is [here](https://en.bitcoin.it/wiki/Segregated_Witness) but what is
more valuable is the list of BIPs (141, 143, 144, and 173) that describe all the technical
protocol upgrades.

You can name this wallet whatever you want...

https://bcoin.io/api-docs/?shell--cli#create-a-wallet

```
bwallet-cli mkwallet --id=test1 --witness=true
```

If this worked you should be able to get an address from the new wallet:

```
bwallet-cli --id=test1 --account=default address
```

Now let's send a transaction from the first wallet with the mined coins to this new wallet:
https://bcoin.io/api-docs/?shell--cli#send-a-transaction

_Note: If you do not pass an `--id=...` option to `bwallet-cli` it will execute the command
with the `default` wallet (the wallet bcoin created on startup)._

```
bwallet-cli send <your new witness address> 10.12345678
```

Did it work?

```
bwallet-cli --id=test1 balance
```

Now confirm that transaction!

```
bcoin-cli rpc generatetoaddress 1 <your address>
```

## Review

We started a full node in regtest, generated coins to a wallet, created a new
wallet using SegWit, and sent coins from one wallet to the other.

Questions:

1. When you requested an address you got a JSON blob with data like `account`, `branch` and `index`.
Do you know what these mean?

2. What did you notice was different about the first wallet address vs. the SegWit address?

3. When you sent the transaction you got a big JSON blob back with data like `fee`, `rate`, and `path`.
What do these mean?

4. When you checked your wallet balance there is `confirmed` and `unconfirmed`.
What's the difference? (bonus: how does bcoin use the terms differently than Bitcoin Core?)

## Homework

1. Look through the API docs and find one or two other commands to experiment with.
Are they RPC commands or REST commands? Can you execute the same command with both `curl`
and the CLI tool?

2. Can you "restore your wallet from seed?" First stop the bcoin node, then delete
your wallet database (!) with `rm -rf ~/.bcoin/regtest/wallet`. When you start bcoin
again can you figure out how to use `mkwallet` to restore your wallet using the BIP39
seed phrase? Once you've restored the wallet, is the balance correct? If not, why not?
(hint: find `rescan` in the API docs).
