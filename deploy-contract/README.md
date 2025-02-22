# Bodhi.js example: Contract Interaction

A basic example on how to sign Acala EVM+ transactions with a Polkadot wallet and the
[bodhi.js](https://github.com/AcalaNetwork/bodhi.js/) SDK.

**INFO: You can find a hosted version of this example here:
[https://bodhi-example-contract.vercel.app/](https://bodhi-example-contract.vercel.app/)**

## Setup

In order to be able to focus on the benefits of using `bodhi.js` in the dApp development, we will be
using `vite` build tool and its `react-ts` template:

````shell
yarn create vite deploy-contract --template react-ts
````

This will create a `deploy-contract` directory with a simple `vite` template with TypeScript
support.

With the skeleton of our dApp ready, it can be modified to use `bodhi.js` to interact with the Acala
EVM+. In order to be able to use it in the dApp, it needs to be added to the project:

````shell
yarn add @acala-network/bodhi
````

Additionally `@polkadot/extension-dapp` is needed for the dApp to be able to retrieve all of the
providers added to the page:

````shell
yarn add @polkadot/extension-dapp
````

To have easy access to the components for the dApp, `antd` is required. Styling is handled by `sass`
and `eslit` is used for linting:

````shell
yarn add antd
yarn add --dev sass eslint
````

`eslint` needs to be initiated:

````shell
yarn eslint --init
````

The example project uses the following options for `eslit` initialization:

````shell
Need to install the following packages:
  @eslint/create-config
Ok to proceed? (y) y
✔ How would you like to use ESLint? · problems
✔ What type of modules does your project use? · esm
✔ Which framework does your project use? · react
✔ Does your project use TypeScript? · No / Yes
✔ Where does your code run? · browser
✔ What format do you want your config file to be in? · JavaScript
The config that you've selected requires the following dependencies:

eslint-plugin-react@latest @typescript-eslint/eslint-plugin@latest @typescript-eslint/parser@latest
✔ Would you like to install them now? · No / Yes
✔ Which package manager do you want to use? · yarn
````

To use the same linting configuration, replace the autogenerated `.eslintrc.cjs` with the
[`eslintrc.js`](./.eslintrc.js).

Once the setup is complete, add the build target to the [`vite.config.ts`](./vite.config.ts):

````typescript
  build: {
    target: 'ESNext',
  }
````

So the complete `vite` configuration should look like this:

````typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vitejs.dev/config/
export default defineConfig({
  build: {
    target: 'ESNext',
  },
  plugins: [react()]
})
````

## Building the dApp

The example dApp will use the
[Echo example's smart contract](https://github.com/AcalaNetwork/hardhat-tutorials/blob/master/echo/contracts/Echo.sol),
deploy it and allow us to update the string stored within it. This requires its compiled version,
[`Echo.json`](./src/Echo.json), to be added to the `src/` directory.

To get the dApp to look as similar as possible to the example one, replace the `src/App.css` with
[`src/App.scss`](./src/App.scss) from the example repository. The `index.css` autogenerated file is
redundant and can be removed. Once it is removed, a reference of it has to be removed from
[`src/main.tsx`](./src/main.tsx). Once you open it, remove the following line:

````tsx
import './index.css'
````

The [`favicon`](./src/favicon.ico) used by the example can also be copied to the `src/` directory.

### Imports and definitions

The autogenerated `App.tsx` can be cleaned out to only include the definition of `App` and its
export:

````tsx
function App() {
  
}

export default App
````

At the top of the file, we have to define the imports. Notable imports are `Provider` and `Signer`
from `@acala-network/bodhi` as well as `WsProvider` from `@polkadot/api` and `web3Example` from
`@polkadot/extension-dapp`. Additionally we have to import the `Echo.json` that we added to the
project earlier as well as the styles contained in the `App.scss`:

````tsx
import React, {
  useCallback, useEffect, useMemo, useState,
} from 'react';
import { Provider, Signer } from '@acala-network/bodhi';
import { WsProvider } from '@polkadot/api';
import { web3Enable } from '@polkadot/extension-dapp';
import type {
  InjectedExtension,
  InjectedAccount,
} from '@polkadot/extension-inject/types';
import { ContractFactory, Contract } from 'ethers';
import { Input, Button, Select } from 'antd';
import echoContract from './Echo.json';

import './App.scss';

const { Option } = Select;

const Check = () => (<span className='check'>✓</span>);
````

We can start populating the `App` function's states by defining the extensions, status flags and
data:

````tsx
  /* ---------- extensions ---------- */
  const [extensionList, setExtensionList] = useState<InjectedExtension[]>([]);
  const [curExtension, setCurExtension] = useState<InjectedExtension | undefined>(undefined);
  const [accountList, setAccountList] = useState<InjectedAccount[]>([]);

  /* ---------- status flags ---------- */
  const [connecting, setConnecting] = useState(false);
  const [loadingAccount, setLoadingAccountInfo] = useState(false);
  const [deploying, setDeploying] = useState(false);
  const [calling, setCalling] = useState(false);

  /* ---------- data ---------- */
  const [provider, setProvider] = useState<Provider | null>(null);
  const [selectedAddress, setSelectedAddress] = useState<string>('');
  const [claimedEvmAddress, setClaimedEvmAddress] = useState<string>('');
  const [balance, setBalance] = useState<string>('');
  const [deployedAddress, setDeployedAddress] = useState<string>('');
  const [echoInput, setEchoInput] = useState<string>('calling an EVM+ contract with polkadot wallet!');
  const [echoMsg, setEchoMsg] = useState<string>('');
  const [newEchoMsg, setNewEchoMsg] = useState<string>('');
  const [url, setUrl] = useState<string>('wss://mandala-rpc.aca-staging.network/ws');
  // const [url, setUrl] = useState<string>('ws://localhost:9944');
````

**On the last line of data definition, the setting for localhost use is commented out. If you want
to use the dApp in the local enviornment, you can comment out the line above it and uncomment the
last line.**

### Connecting to the chain node with the provider

Connecting the provider to the chain node allows us to use the provider to communicate with the
chain:

````tsx
  /* ------------ Step 1: connect to chain node with a provider ------------ */
  const connectProvider = useCallback(async (nodeUrl: string) => {
    setConnecting(true);
    try {
      const signerProvider = new Provider({
        provider: new WsProvider(nodeUrl.trim()),
      });

      await signerProvider.isReady();

      setProvider(signerProvider);
    } catch (error) {
      console.error(error);
      setProvider(null);
    } finally {
      setConnecting(false);
    }
  }, []);
````

### Connect to the Polkadot wallet

Connecting the dApp to the Polkadot wallet allows users to sign the messages:

````tsx
  /* ------------ Step 2: connect polkadot wallet ------------ */
  const connectWallet = useCallback(async () => {
    const allExtensions = await web3Enable('bodhijs-example');
    setExtensionList(allExtensions);
    setCurExtension(allExtensions[0]);
  }, []);

  useEffect(() => {
    curExtension?.accounts.get().then(result => {
      setAccountList(result);
      setSelectedAddress(result[0].address || '');
    });
  }, [curExtension]);
````

This will prompt all of the installed extension of the browser to connect to the dApp. User has the
ability to only connect one or all of them. Connecting more than one wallet will give users the
ability to pick which wallet to use to sign the transactions.

With the access to the accounts in the Polkadot wallet, we can use that to create a `Signer` object:

````tsx
  /* ----------
     Step 2.1: create a bodhi signer from provider and extension signer
                                                             ---------- */
  const signer = useMemo(() => {
    if (!provider || !curExtension || !selectedAddress) return null;
    return new Signer(provider, selectedAddress, curExtension.signer);
  }, [provider, curExtension, selectedAddress]);
````

Now that we have access to the `Signer` as well as the network, we can return additional information
about the `Signer`. Some information that we can present are the bound EVM addresses and the
balances of the `Signer`s:

````tsx
  /* ----------
     Step 2.2: locad some info about the account such as:
     - bound/default evm address
     - balance
     - whatever needed
                                               ---------- */
  useEffect(() => {
    (async function fetchAccountInfo() {
      if (!signer) return;

      setLoadingAccountInfo(true);
      try {
        const [evmAddress, accountBalance] = await Promise.all([
          signer.queryEvmAddress(),
          signer.getBalance(),
        ]);
        setBalance(accountBalance.toString());
        setClaimedEvmAddress(evmAddress);
      } catch (error) {
        console.error(error);
        setClaimedEvmAddress('');
        setBalance('');
      } finally {
        setLoadingAccountInfo(false);
      }
    }());
  }, [signer]);
````

### Deploy the smart contract

With the `Signer`ready, we can finally deploy the `Echo` smart contract:

````tsx
  /* ------------ Step 3: deploy contract ------------ */
  const deploy = useCallback(async () => {
    if (!signer) return;

    setDeploying(true);
    try {
      const factory = new ContractFactory(echoContract.abi, echoContract.bytecode, signer);

      const contract = await factory.deploy();
      const echo = await contract.echo();

      setDeployedAddress(contract.address);
      setEchoMsg(echo);
    } finally {
      setDeploying(false);
    }
  }, [signer]);
````

In addition to deploying the smart contract, we are also returning the address of it once it is
deployed and retrieving the `string` stored within it.

### Interact with the smart contract

The `Echo` smart contract allows for changing the `string` stored within it. We can add the changing
of it to our dApp:

````tsx
  /* ------------ Step 4: call contract ------------ */
  const callContract = useCallback(async (msg: string) => {
    if (!signer) return;
    setCalling(true);
    setNewEchoMsg('');
    try {
      const instance = new Contract(deployedAddress, echoContract.abi, signer);

      await instance.scream(msg);
      const newEcho = await instance.echo();

      setNewEchoMsg(newEcho);
    } finally {
      setCalling(false);
    }
  }, [signer, deployedAddress]);
````

### Add utilities

In addition to the functionalities that we added above, we also require the ability for users to
select the wallet extension that they want to use with the dApp and the ability for them to select
an account from that extension:

````tsx
  // eslint-disable-next-line
  const ExtensionSelect = () => (
    <div>
      <span style={{ marginRight: 10 }}>select a polkadot wallet:</span>
      <Select
        value={ curExtension?.name }
        onChange={ targetName => setCurExtension(extensionList.find(e => e.name === targetName)) }
        disabled={ !!deployedAddress }
      >
        {extensionList.map(ex => (
          <Option key={ ex.name } value={ ex.name }>
            {`${ex.name}/${ex.version}`}
          </Option>
        ))}
      </Select>
    </div>
  );

  // eslint-disable-next-line
  const AccountSelect = () => (
    <div>
      <span style={{ marginRight: 10 }}>account:</span>
      <Select
        value={ selectedAddress }
        onChange={ value => setSelectedAddress(value) }
        disabled={ !!deployedAddress }
      >
        {accountList.map(account => (
          <Option key={ account.address } value={ account.address }>
            {account.name} / {account.address}
          </Option>
        ))}
      </Select>
    </div>
  );
````

### Build the interface

All of the functionality of our dApp is ready to be presented in the interface. We will wrap the
interface in a `return` statement as we want our `App` function to return it:

````tsx
  return (
    <div id='app'>

    </div>
  );
````

In the first section of the interface, we will utilize the chain node connection mechanic:

````tsx
      { /* ------------------------------ Step 1 ------------------------------*/ }
      <section className='step'>
        <div className='step-text'>Step 1: Connect Chain Node { provider && <Check /> }</div>
        <Input
          type='text'
          disabled={ connecting || !!provider }
          value={ url }
          onChange={ e => setUrl(e.target.value) }
          addonBefore='node url'
        />
        <Button
          type='primary'
          onClick={ () => connectProvider(url) }
          disabled={ connecting || !!provider }
        >
          { connecting
            ? 'connecting ...'
            : provider
              ? `connected to ${provider.api.runtimeChain.toString()}`
              : 'connect' }
        </Button>
      </section>
````

This will allow users to use the default chain node or input their own.

In the second section we will use the mechainc to connect to the wallet extensions of the user's
browser and use the utilities to select the preferred wallet extension and preferred account for the
interaction with the chain. It will also provide the information about the connected account:

````tsx
      { /* ------------------------------ Step 2 ------------------------------*/}
      <section className='step'>
        <div className='step-text'>Step 2: Connect Polkadot Wallet { signer && <Check />  }</div>
        <div>
          <Button
            type='primary'
            onClick={ connectWallet }
            disabled={ !provider || !!signer }
          >
            {curExtension
              ? `connected to ${curExtension.name}/${curExtension.version}`
              : 'connect'}
          </Button>

          { !!extensionList?.length && <ExtensionSelect /> }
          { !!accountList?.length && <AccountSelect /> }
        </div>

        {signer && (
          <div>
            {loadingAccount
              ? 'loading account info ...'
              : claimedEvmAddress
                ? (<div>claimed evm address: <span className='address'>{claimedEvmAddress}</span></div>)
                : (<div>default evm address: <span className='address'>{signer.computeDefaultEvmAddress()}</span></div>)}
            { balance && (<div>account balance: <span className='address'>{ balance }</span></div>) }
          </div>
        )}
      </section>
````

The third section will allow users to deploy the smart contract and present its values once it is
included into the chain:

````tsx
      { /* ------------------------------ Step 3 ------------------------------*/}
      <section className='step'>
        <div className='step-text'>Step 3: Deploy Echo Contract { deployedAddress && <Check /> }</div>
        <Button
          type='primary'
          disabled={ !signer || deploying || !!deployedAddress }
          onClick={ deploy }
        >
          { deployedAddress
            ? 'contract deployed'
            : deploying
              ? 'deploying ...'
              : 'deploy'}
        </Button>

        {deployedAddress && (
          <>
            <div>contract address: <span className='address'>{deployedAddress}</span></div>
            <div>initial echo messge: <span className='address'>{echoMsg}</span></div>
          </>
        )}
      </section>
````

The final section enables the user to interact with the `Echo` smart contract deployed in the
previous section in order to change the `string` stored within it. It will also present the updated
`string` once it's added to the chain and include the congratulatory message, notifying users of
successful completion of the example flow:

````tsx
      { /* ------------------------------ Step 4 ------------------------------*/}
      <section className='step'>
        <div className='step-text'>Step 4: Call Contract To Change Echo Msg { newEchoMsg && <Check /> }</div>
        <Input
          type='text'
          disabled={ !signer || !deployedAddress || calling }
          value={ echoInput }
          onChange={ e => setEchoInput(e.target.value) }
          addonBefore='new msg'
        />
        <Button
          type='primary'
          disabled={ !signer || !deployedAddress || calling }
          onClick={ () => callContract(echoInput) }
        >
          { calling
            ? 'sending tx ...'
            : 'call'}
        </Button>

        {newEchoMsg && (
          <div>new echo messge: <span className='address'>{newEchoMsg}</span></div>
        )}
      </section>

      {newEchoMsg && (
        <section className='step' id='congrats'>
          <div>Congratulations 🎉🎉</div>
          <div>You have succesfully deployed and called an EVM+ contract with <span className='cross'>metamask</span><span className='decorate'>polkadot wallet</span></div>
          <Button
            id='next-level'
            type='primary'
            onClick={ () => window.open('https://github.com/AcalaNetwork/bodhi-examples/tree/master/batch-transactions', '_blank') }
          >
            Take Me To Advanced Example
          </Button>
        </section>
      )}
````

This conclues our example dApp that showcases how a Polkadot wallet can be used to interact with the
Acala EVM+.

<details>
    <summary>Your src/App.tsx should look like this:</summary>

    import React, {
        useCallback, useEffect, useMemo, useState,
    } from 'react';
    import { Provider, Signer } from '@acala-network/bodhi';
    import { WsProvider } from '@polkadot/api';
    import { web3Enable } from '@polkadot/extension-dapp';
    import type {
        InjectedExtension,
        InjectedAccount,
    } from '@polkadot/extension-inject/types';
    import { ContractFactory, Contract } from 'ethers';
    import { Input, Button, Select } from 'antd';
    import echoContract from './Echo.json';

    import './App.scss';

    const { Option } = Select;

    const Check = () => (<span className='check'>✓</span>);

    function App() {
        /* ---------- extensions ---------- */
        const [extensionList, setExtensionList] = useState<InjectedExtension[]>([]);
        const [curExtension, setCurExtension] = useState<InjectedExtension | undefined>(undefined);
        const [accountList, setAccountList] = useState<InjectedAccount[]>([]);

        /* ---------- status flags ---------- */
        const [connecting, setConnecting] = useState(false);
        const [loadingAccount, setLoadingAccountInfo] = useState(false);
        const [deploying, setDeploying] = useState(false);
        const [calling, setCalling] = useState(false);

        /* ---------- data ---------- */
        const [provider, setProvider] = useState<Provider | null>(null);
        const [selectedAddress, setSelectedAddress] = useState<string>('');
        const [claimedEvmAddress, setClaimedEvmAddress] = useState<string>('');
        const [balance, setBalance] = useState<string>('');
        const [deployedAddress, setDeployedAddress] = useState<string>('');
        const [echoInput, setEchoInput] = useState<string>('calling an EVM+ contract with polkadot wallet!');
        const [echoMsg, setEchoMsg] = useState<string>('');
        const [newEchoMsg, setNewEchoMsg] = useState<string>('');
        const [url, setUrl] = useState<string>('wss://mandala-rpc.aca-staging.network/ws');
        // const [url, setUrl] = useState<string>('ws://localhost:9944');

        /* ------------ Step 1: connect to chain node with a provider ------------ */
        const connectProvider = useCallback(async (nodeUrl: string) => {
            setConnecting(true);
            try {
                const signerProvider = new Provider({
                    provider: new WsProvider(nodeUrl.trim()),
                });

                await signerProvider.isReady();

                setProvider(signerProvider);
            } catch (error) {
                console.error(error);
                setProvider(null);
            } finally {
                setConnecting(false);
            }
        }, []);

        /* ------------ Step 2: connect polkadot wallet ------------ */
        const connectWallet = useCallback(async () => {
            const allExtensions = await web3Enable('bodhijs-example');
            setExtensionList(allExtensions);
            setCurExtension(allExtensions[0]);
        }, []);

        useEffect(() => {
            curExtension?.accounts.get().then(result => {
                setAccountList(result);
                setSelectedAddress(result[0].address || '');
            });
        }, [curExtension]);

        /* ----------
            Step 2.1: create a bodhi signer from provider and extension signer
                                                                    ---------- */
        const signer = useMemo(() => {
            if (!provider || !curExtension || !selectedAddress) return null;
            return new Signer(provider, selectedAddress, curExtension.signer);
        }, [provider, curExtension, selectedAddress]);

        /* ----------
            Step 2.2: locad some info about the account such as:
            - bound/default evm address
            - balance
            - whatever needed
                                                    ---------- */
        useEffect(() => {
            (async function fetchAccountInfo() {
                if (!signer) return;

                setLoadingAccountInfo(true);
                try {
                    const [evmAddress, accountBalance] = await Promise.all([
                        signer.queryEvmAddress(),
                        signer.getBalance(),
                    ]);
                    setBalance(accountBalance.toString());
                    setClaimedEvmAddress(evmAddress);
                } catch (error) {
                    console.error(error);
                    setClaimedEvmAddress('');
                    setBalance('');
                } finally {
                    setLoadingAccountInfo(false);
                }
            }());
        }, [signer]);

        /* ------------ Step 3: deploy contract ------------ */
        const deploy = useCallback(async () => {
            if (!signer) return;

            setDeploying(true);
            try {
                const factory = new ContractFactory(echoContract.abi, echoContract.bytecode, signer);

                const contract = await factory.deploy();
                const echo = await contract.echo();

                setDeployedAddress(contract.address);
                setEchoMsg(echo);
            } finally {
                setDeploying(false);
            }
        }, [signer]);

        /* ------------ Step 4: call contract ------------ */
        const callContract = useCallback(async (msg: string) => {
            if (!signer) return;
            setCalling(true);
            setNewEchoMsg('');
            try {
                const instance = new Contract(deployedAddress, echoContract.abi, signer);

                await instance.scream(msg);
                const newEcho = await instance.echo();

                setNewEchoMsg(newEcho);
            } finally {
                setCalling(false);
            }
        }, [signer, deployedAddress]);

        // eslint-disable-next-line
        const ExtensionSelect = () => (
            <div>
                <span style={{ marginRight: 10 }}>select a polkadot wallet:</span>
                <Select
                    value={ curExtension?.name }
                    onChange={ targetName => setCurExtension(extensionList.find(e => e.name === targetName)) }
                    disabled={ !!deployedAddress }
                >
                    {extensionList.map(ex => (
                        <Option key={ ex.name } value={ ex.name }>
                            {`${ex.name}/${ex.version}`}
                        </Option>
                    ))}
                </Select>
            </div>
        );

        // eslint-disable-next-line
        const AccountSelect = () => (
            <div>
                <span style={{ marginRight: 10 }}>account:</span>
                <Select
                    value={ selectedAddress }
                    onChange={ value => setSelectedAddress(value) }
                    disabled={ !!deployedAddress }
                >
                    {accountList.map(account => (
                        <Option key={ account.address } value={ account.address }>
                            {account.name} / {account.address}
                        </Option>
                    ))}
                </Select>
            </div>
        );

        return (
            <div id='app'>
                { /* ------------------------------ Step 1 ------------------------------*/ }
                <section className='step'>
                    <div className='step-text'>Step 1: Connect Chain Node { provider && <Check /> }</div>
                    <Input
                        type='text'
                        disabled={ connecting || !!provider }
                        value={ url }
                        onChange={ e => setUrl(e.target.value) }
                        addonBefore='node url'
                    />
                    <Button
                        type='primary'
                        onClick={ () => connectProvider(url) }
                        disabled={ connecting || !!provider }
                    >
                        { connecting
                            ? 'connecting ...'
                            : provider
                                ? `connected to ${provider.api.runtimeChain.toString()}`
                                : 'connect' }
                    </Button>
                </section>

                { /* ------------------------------ Step 2 ------------------------------*/}
                <section className='step'>
                    <div className='step-text'>Step 2: Connect Polkadot Wallet { signer && <Check />  }</div>
                    <div>
                        <Button
                            type='primary'
                            onClick={ connectWallet }
                            disabled={ !provider || !!signer }
                        >
                            {curExtension
                            ? `connected to ${curExtension.name}/${curExtension.version}`
                            : 'connect'}
                        </Button>

                        { !!extensionList?.length && <ExtensionSelect /> }
                        { !!accountList?.length && <AccountSelect /> }
                    </div>

                    {signer && (
                        <div>
                            {loadingAccount
                                ? 'loading account info ...'
                                : claimedEvmAddress
                                    ? (<div>claimed evm address: <span className='address'>{claimedEvmAddress}</span></div>)
                                    : (<div>default evm address: <span className='address'>{signer.computeDefaultEvmAddress()}</span></div>)}
                            { balance && (<div>account balance: <span className='address'>{ balance }</span></div>) }
                        </div>
                    )}
                </section>

                { /* ------------------------------ Step 3 ------------------------------*/}
                <section className='step'>
                    <div className='step-text'>Step 3: Deploy Echo Contract { deployedAddress && <Check /> }</div>
                    <Button
                        type='primary'
                        disabled={ !signer || deploying || !!deployedAddress }
                        onClick={ deploy }
                    >
                    { deployedAddress
                        ? 'contract deployed'
                        : deploying
                            ? 'deploying ...'
                            : 'deploy'}
                    </Button>

                    {deployedAddress && (
                       <>
                            <div>contract address: <span className='address'>{deployedAddress}</span></div>
                            <div>initial echo messge: <span className='address'>{echoMsg}</span></div>
                        </>
                    )}
                </section>

                { /* ------------------------------ Step 4 ------------------------------*/}
                <section className='step'>
                    <div className='step-text'>Step 4: Call Contract To Change Echo Msg { newEchoMsg && <Check /> }</div>
                    <Input
                        type='text'
                        disabled={ !signer || !deployedAddress || calling }
                        value={ echoInput }
                        onChange={ e => setEchoInput(e.target.value) }
                        addonBefore='new msg'
                    />
                    <Button
                        type='primary'
                        disabled={ !signer || !deployedAddress || calling }
                        onClick={ () => callContract(echoInput) }
                    >
                    { calling
                        ? 'sending tx ...'
                        : 'call'}
                    </Button>

                    {newEchoMsg && (
                        <div>new echo messge: <span className='address'>{newEchoMsg}</span></div>
                    )}
                </section>

                {newEchoMsg && (
                    <section className='step' id='congrats'>
                        <div>Congratulations 🎉🎉</div>
                        <div>You have succesfully deployed and called an EVM+ contract with <span className='cross'>metamask</span><span className='decorate'>polkadot wallet</span></div>
                        <Button
                            id='next-level'
                            type='primary'
                            onClick={ () => window.open('https://github.com/AcalaNetwork/bodhi-examples/tree/master/batch-transactions', '_blank') }
                        >
                            Take Me To Advanced Example
                        </Button>
                    </section>
                )}
            </div>
        );
    }

    export default App;

</details>

## Running the example

Initializing `vite` template already added the `preview`, which is used to run the preview of the
dApp, and `build` scripts. The build command is used to build the dApp:

````shell
yarn build
yarn run v1.22.19
$ tsc && vite build
vite v3.0.2 building for production...
✓ 4099 modules transformed.
dist/index.html                  0.45 KiB
dist/assets/index.c0ece840.css   545.01 KiB / gzip: 66.58 KiB
dist/assets/index.9912a485.js    3268.36 KiB / gzip: 1044.98 KiB

(!) Some chunks are larger than 500 KiB after minification. Consider:
- Using dynamic import() to code-split the application
- Use build.rollupOptions.output.manualChunks to improve chunking: https://rollupjs.org/guide/en/#outputmanualchunks
- Adjust chunk size limit for this warning via build.chunkSizeWarningLimit.
✨  Done in 15.99s.
````

Running `yarn preview` should return the following output:

````shell
yarn preview
yarn run v1.22.19
$ vite preview
  ➜  Local:   http://localhost:4173/
  ➜  Network: use --host to expose
````

Visiting the URL specified in the outout should return the locall address where you can see the
dApp:

![dApp preview](./img/preview.png)

We can also add a `lint` script to the `package.json` in order to lint the dApp:

````json
    "lint": "tsc --noEmit; eslint . --ext .js,.jsx,.ts,.tsx"
````

Your dApp should now be ready for users to deploy the example `Echo` smart contract to the Acala
EVM+ using a Polkadot wallet. Happy coding!