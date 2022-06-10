# @kwe-defi/evm-provider.js

`evm-provider.js` implements a web3 provider which can interact with the [Kwe chain EVM](https://github.com/kwe-defi/kwe-chain).

If you only care about developing Solidity contracts on the Kwe chain, `@kwe-defi/evm-provider.js` is used in our [Hardhat Kwe environment](https://github.com/kwe-defi/hardhat-kwe). The environment simplifies and abstracts all the low-level intricacies, so you can only focus on the Solidity part. See [hardhat-kwe-examples repo](https://github.com/kwe-defi/hardhat-kwe-examples/blob/master/scripts/flipper/deploy.js) for more examples.

If you need more control, then it can also be used as a Substrate provider to query or to interact with the Kwe chain using the same calls as in the [Polkadot.js](https://polkadot.js.org/docs/api).

## Installation

Install dependencies with `yarn` [see issue](https://github.com/kwe-defi/evm-provider.js/issues/5#issuecomment-912389541).

### Yarn

```bash
yarn add @kwe-defi/evm-provider
```

## Getting started

To create a `Provider` instance, use the following code:

```javascript
import {
  TestAccountSigningKey,
  Provider,
  Signer,
} from "@kwe-defi/evm-provider";
import { WsProvider, Keyring } from "@polkadot/api";
import { createTestPairs } from "@polkadot/keyring/testingPairs";
import { KeyringPair } from "@polkadot/keyring/types";

const WS_URL = process.env.WS_URL || "ws://127.0.0.1:9944";
const seed = process.env.SEED;

const setup = async () => {
  const provider = new Provider({
    provider: new WsProvider(WS_URL),
  });

  await provider.api.isReady;

  let pair: KeyringPair;
  if (seed) {
    const keyring = new Keyring({ type: "sr25519" });
    pair = keyring.addFromUri(seed);
  } else {
    const testPairs = createTestPairs();
    pair = testPairs.alice;
  }

  const signingKey = new TestAccountSigningKey(provider.api.registry);
  signingKey.addKeyringPair(pair);

  const signer = new Signer(provider, pair.address, signingKey);

  // Claim default account
  if (!(await signer.isClaimed())) {
    console.log(
      "No claimed EVM account found -> claimed default EVM account: ",
      await signer.getAddress()
    );
    await signer.claimDefaultAccount();
  }

  return {
    signer,
    provider,
  };
};

export default setup;
```

with this object you can interact with the Substrate chain.

If you want to interact with injected sources (e.g. from Polkadot{.js}) you can do the following:

```javascript
import { Provider, Signer, } from "@kwe-defi/evm-provider";
import { WsProvider } from "@polkadot/api";
import { web3Accounts, web3Enable } from "@polkadot/extension-dapp";

const WS_URL = process.env.WS_URL || "ws://127.0.0.1:9944";
const seed = process.env.SEED;

const setup = async () => {
  
  // Return an array of all the injected sources
  // (this needs to be called first)
  const allInjected = await web3Enable('your dapp');

  const injected;
  if (allInjected[0] && allInjected[0].signer) {
    injected = allInjected[0].signer;
  }

  // Return an array of { address, meta: { name, source } }
  // (meta.source contains the name of the extension)
  const allAccounts = await web3Accounts();

  let account;
  if (allAccounts[0] && allAccounts[0].address) {
    account = allAccounts[0].address;
  }

  const provider = new Provider({
    provider: new WsProvider(WS_URL)
  });

  await provider.api.isReady;

  const signer = new Signer(provider, account, injected);

  // Claim default account
  if (!(await signer.isClaimed())) {
    console.log(
      "No claimed EVM account found -> claimed default EVM account: ",
      await signer.getAddress()
    );
    await signer.claimDefaultAccount();
  }

  return {
    signer,
    provider,
  };
};

export default setup;
```

## EVM interaction

Most, but not all, of `evm-provider.js` API is compatible with `ethers.js`. If you are not familiar with `ethers.js`, you can start by looking at its [documentation](https://docs.ethers.io/v5/single-page/). See our [Kweswap example](https://github.com/kwe-defi/kweswap/blob/653e6f4e77d228bba32fe233bff4a4811eae335e/src/deploy.ts) on how it uses the above `setup` script to deploy and interact with the EVM.

### Get EVM address

```javascript
// ethers
let accounts = await this.provider.listAccounts();
let selectedAccount = accounts[0];

// evm-provider
let selectedAccount = await this.signer.queryEvmAddress();
```

### Claim EVM address

If you would like to inject an evm address that you already own you can do so via the `claimAccount` extrinsic.  The script below illustrates how this can be done.

```javascript
import { Keyring, WsProvider } from '@polkadot/api';
import { ethers } from 'ethers';
import { createClaimEvmSignature } from './utils';
import { Provider } from '.';

const WS_URL = process.env.WS_URL || 'ws://127.0.0.1:9944';
// kwe address - 5H728gLgx4yuCSVEwGCAfLo3RtzTau9F6cTNqNJtrqqjACWq
const kwePrivKeyRaw = process.env.KWE_PRIV_KEY || "0x0000000000000000000000000000000000000000000000000000000000000000";
const ethPrivKey = process.env.ETH_PRIV_KEY || "0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a";

const main = async (): Promise<void> =>  {
    const provider = new Provider({
        provider: new WsProvider(WS_URL)
    });
    await provider.api.isReady;

    const keyring = new Keyring({ type: 'sr25519' });
    const kweKey = keyring.addFromUri(kwePrivKeyRaw);
    const ethKey = new ethers.Wallet(ethPrivKey);

    const msg = createClaimEvmSignature(kweKey.address);
    let signature = await ethKey.signMessage(msg);

    await provider.api.tx.evmAccounts.claimAccount(
        ethKey.address,
        signature
    ).signAndSend(kweKey);

    process.exit();
};

main();
```

### Provider

The Provider provides an API for interacting with nodes and is an instance of `ethers.js` [AbstractProvider](https://docs.ethers.io/v5/single-page/#/v5/api/providers/-%23-providers).

### Signer

The Signer class can sign transactions and messages using a private key. When using the wallet for the first time, make sure to always claim the EVM account for the wallet you are using:

```javascript
signer.claimDefaultAccount();
```

before performing any EVM calls otherwise it may lead to `InsufficientBalance` errors.

## Gas limit and storage limit
In addition to the gas limit (processing), the Kwe chain also charges a [storage fee](https://docs.substrate.io/v3/runtime/smart-contracts/#storage-deposit). When you interact with the EVM, Kwe chain will estimate both fees and as such the fees will be invisible to the user. This should work in 99% of cases. It assumes you have at least 60 KWE tokens on the signing account. However, sometimes the heuristics (usually for more complex contracts) are wrong. In this case you can force the values of `gasLimit` and `storageLimit` by adding them to the `options` dictionary at the end of every call, for example:

```
await factory.deploy(<contract_args>, {
  gasLimit: 1000000,
  customData: { storageLimit: 1000000 }
});
```

If you require maximum flexibility `evm-provider` exports maximum gas and storage limit:

```
import { MAX_GAS_LIMIT, MAX_STORAGE_LIMIT } from "@kwe-defi/evm-provider";
```
which default to `U64MAX` and `U32MAX` respectively.

## Versions
- versions 1.\*.\* work from Kwe v8 chain onwards
  - no longer requires `resolutions` with `ethers@5.0.9`
- versions 0.\*.\* work from Kwe v0 to v7

#### [Changelog](./CHANGELOG.md)