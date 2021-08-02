# Week 4: bech32m

This week we are going to start on a new feature for bcoin: bech32m addresses.
The goal is to extend the `Address` module to _send_ to bech32m addresses,
which will be used for the first time for Taproot.

## Background

Bitcoin has historically had several [address types](https://en.bitcoin.it/wiki/Invoice_address):

- pay-to-public-key (p2pk): This wasn't even an address format but in the very
early days, transaction output scripts contained a raw public key. Satoshi Nakamoto's
famous first transaction to Hal Finney is a good example: https://blockstream.info/tx/0437cd7f8525ceed2324359c2d0ba26006d92d856a9c20fa0241106ee5a597c9?expand

- pay-to-pubic-key-hash (p2pkh): Using Base58check encoding:
  - https://en.bitcoin.it/Base58Check_encoding
  - https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses

- pay-to-script-hash (p2sh): Using Base58check encoding:
  - https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki

- pay-to-witness-public-key-hash (p2wpkh): Using Bech32 encoding:
  - https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#p2wpkh

- pay-to-witness-script-hash (p2wsh): Using Bech32 encoding:
  - https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#P2WSH

- pay-to-taproot (p2tr): Using Bech32m encoding:
  - https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki


## How do Bitcoin addresses work?

- The address itself never appears on the blockchain, it is merely a UI element.
- The recipient generates an address with their wallet.
- The sender enters the address into their wallet, which creates a transaction
output constructed exactly how the recipient wants it, so that they can spend
it in the future.
- The sender's wallet DOES NOT need to be able to spend from this address type.
  - This means that after your work is done, bcoin will be able to send money
  to taproot recipients, even though taproot functionality is not implemented in
  bcoin itself.

### Examples:

Base58 p2pkh:
  - address: `1Ak8PffB2meyfYnbXZR9EGfLfFZVpzJvQP`
  - output script: `OP_DUP OP_HASH160 6ae1301cf44ca525751d1763ac4fef12d1153986 OP_EQUALVERIFY OP_CHECKSIG`

Base58 p2sh:
  - address: `3Ftj6zuXhYaWmSC7jtsrJPtADtiWb9LUF2`
  - output script: `OP_0 1c7b17c92ebb9171ba06877fe92acff2a6301f39`

bech32 p2pkh:
  - address: `bc1qjn3gredclhz3eqprlsuh4js066ygnnxmwmmeyu`
  - output script: `OP_0 94e281e5b8fdc51c8023fc397aca0fd68889ccdb`

bech32m p2tr:
  - address: `bc1p0xlxvlhemja6c4dqv22uapctqupfhlxm9h8z3k2e72q4k9hcz7vqzk5jj0`
  - output script: `OP_1 79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`

## How does bcoin parse addresses?

THe important modules to review are
[`address.js`](https://github.com/bcoin-org/bcoin/blob/master/lib/primitives/address.js)
and
[`address-test.js`](https://github.com/bcoin-org/bcoin/blob/master/test/address-test.js)

These two files aren't that long and to get in to this project I recommend reading
them from top to bottom. Run the tests, try to break the tests by changing parts of
code, etc so you understand what these functions are doing.

## bcrypto

Notice at the top of `address.js` we actually require two modules from a dependency
for the actual base58 and bech32 encoding:

```
const base58 = require('bcrypto/lib/encoding/base58');
const bech32 = require('bcrypto/lib/encoding/bech32');
```

The [bcrypto library](https://github.com/bcoin-org/bcrypto) is an essential
component of bcoin and part of your work for this project will be a pull request
to that repository, in addition to the bcoin repository.

bcrypto is kind of crazy because everything is implemented in BOTH C and JavaScript.
You'll notice the file `bech32.js` is actually a proxy:

```js
if (process.env.NODE_BACKEND === 'js')
  module.exports = require('../js/bech32');
else
  module.exports = require('../native/bech32');
```

bech32 is implemented in JavaScript in [`lib/js/bech32.js`](https://github.com/bcoin-org/bcrypto/blob/master/lib/js/bech32.js) and in C in [`deps/torsion/src/encoding.c`](https://github.com/bcoin-org/bcrypto/blob/master/deps/torsion/src/encoding.c#L1180)
which is bound to JavaScript in [`lib/native/bech32.js`](https://github.com/bcoin-org/bcrypto/blob/master/lib/native/bech32.js).

## bech32m

What is the difference between bech32 and bech32m? Turns out, it's JUST the checksum:

https://github.com/sipa/bech32/blob/master/ref/javascript/bech32.js#L35-L43

```js
function getEncodingConst (enc) {
  if (enc == encodings.BECH32) {
    return 1;
  } else if (enc == encodings.BECH32M) {
    return 0x2bc830a3;
  } else {
    return null;
  }
}
```

## Project work

We will just be adding bech32m support in JavaScript for now. Here is how I recommend
you proceed:

Start with bcoin master branch and then create a new branch:

```
cd bcoin
git checkout master
git checkout -b bech32m
```

Remove the `bcrpyto` package that came installed with the bcoin repo and re-clone
it from git:

```
cd node_modules
rm -rf bcrypto
git clone https://bcoin-org/bcrypto
cd bcrypto
npm install
git checkout -b bech32m
```

So this may be a bit confusing but keep in mind we have two git repos now one
inside the other!

So since bech32 and bech32m are the exact same algorithm but with different
checksum constants, here is how I propose we do this:

1. Refactor the existing bech32 module as a class with a constructor.
All the functions in that file need to be object methods now.

```js
class bech32{
  constructor(name) {
    this.checksum = null;

    switch (name) {
      'BECH32':
        this.checksum = 1;
        break;
      'BECH32m':
        this.checksum = 0x2bc830a3;
        break;
      default:
        throw new Error('Unknown variant.');
    }
  }
...
}
```

2. Replace the bech32 checksum constant (`1`) with `this.checksum`:

https://github.com/pinheadmz/bcrypto/blob/master/lib/js/bech32.js#L207


3. Refactor the file `lib/encoding/bech32.js` to look more like
[`lib/js/secp256k1.js`](https://github.com/pinheadmz/bcrypto/blob/master/lib/js/secp256k1.js).
What we want to do is `require` the bech32 module, but create a new instance and pass
it the `BECH32` name.

4. This refactor should not have broken the library! Test:

```
export NODE_BACKEND=js
bmocha test/bech32-test.js
```

5. Now we can start integrating bech32m. Create a new file `lib/encoding/bech32m.js`:

```js
'use strict';

const bech32 = require('../js/bech32');

// TODO: Implement bech32m in C as well and bind here.
module.exports = new bech32('BECH32m');
```

6. Add tests: You can edit the file `test/bech32-test.js` and add tests for bech32m.
There are test vectors you can copy in [BIP350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki#Test_vectors_for_Bech32m)
and you can also use Pieter Wuille's [demo website](http://bitcoin.sipa.be/bech32/demo/demo.html).

7. BACK TO BCOIN. Now that we have a bech32m implementation in bcrypto, we need
to integrate it into the bcoin `Address` module - and add tests!

Add this at the top of the file `address.js`:

```js
const bech32m = require('bcrypto/lib/encoding/bech32m');
```

You're on your own from here, but here are some hints:

Take a look at the `decode()` and `encode()` functions in
https://github.com/sipa/bech32/blob/master/ref/javascript/segwit_addr.js

Remember that witness version 0 addresses are always bech32, and any other
version is bech32m.
