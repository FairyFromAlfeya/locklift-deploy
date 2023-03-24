# Locklift-deploy plugin
The plugin provides setup environment features, like deploying contracts and setup accounts.
### Setup plugin
1. `npm i locklift-deploy`
2. Update `locklift.config.ts`
```typescript
import "locklift-deploy";
import { Deployments } from "locklift-deploy";

declare module "locklift" {
    //@ts-ignore
    export interface Locklift {
        deployments: Deployments<FactorySource>;
    }
}

// Default deploments folder is `deploy` but the folder could be changed
const config: LockliftConfig = {
    deployments: {
        pathToDeployLog: "my-deploy-folder",
    },
    //////////////////////
}
```

### Usage
First of all, need to create `deploy folder (it will `deploy` or a folder that was provided inside the config).
Inside this folder, we are going to create our first files let's call them `deploy-sample.ts` and `crate-account.ts`.So our project structure will look like this
```
├── contracts
│   └── Sample.tsol
├── locklift.config.ts
├── scripts
│   └── 1-deploy-sample.ts
├── test
│   └── sample-test.ts
└─── deployments
    ├── deploy-sample.ts
    └── crate-account.ts
```
#### Note deploy files have so particular structure, the file should include:
1. export default function, that is returns the promise
```typescript
export default async () => {
    // we will update the function body letter
};
```
2. tag name
```typescript
export const tag = "sample1";
```
3. And optionally it can include dependencies array
```typescript
export const dependencies = ["sample2", "sample3", "sample4"];
```

#### Implementing deploy behavior for `Sample` contract that locklift generated when the project was initialized (example)
1. Create an account in file `crate-account.ts`
```typescript
export default async () => {
  await locklift.deployments.createAccounts([
    {
      accountName: "Deployer", // custom account name, this name will be used for getting access to the account
      signerId: "0", // locklift.keystore.getSigner("0") <- this is id for getting access to the particular signer
      accountSettings: {
        type: WalletTypes.EverWallet,
        value: toNano(10),
      },
    },
  ]);
};

export const tag = "create-my-account";
```
2. Deploy `Sample contract` with our `Deployer` publicKey
```typescript
export default async () => {
  const INIT_STATE = 0;
    //we got access by our custom name that was provided when we were creating an account
  const deployer = locklift.deployments.getAccount("Deployer");
    // And now we are deploying via `deployments API`
  await locklift.deployments.deploy({
    deployConfig: {
      contract: "Sample",
      publicKey: deployer.signer.publicKey,
      initParams: {
        _nonce: locklift.utils.getRandomNonce(),
      },
      constructorParams: {
        _state: INIT_STATE,
      },
      value: locklift.utils.toNano(2),
    },
    contractName: "Sample1",// custom contract name, this name will be used for getting an access
  });
};

export const tag = "sample1";
// we need to set `create-my-account` as a dependency. 
// It will guarantee that account will be created earlier than this script will be run
export const dependencies = ["create-my-account"]; 
```
3. Let's move to the test folder and try to have access to our `Sample1` contract.
   Note, deployments are lazy. So we need to trigger the deployments flow by using
```typescript
await locklift.deployments.fixture();
```
It will trigger the deployments flow, and after it, we will have access to `Deployer` account and `Sample1` contract
```typescript
const sample = locklift.deployments.getContract<SampleAbi>("Sample1");
// Generic should be setted ----------------------^ // it can be found inside the factorySource.ts
const deployer = locklift.deployments.getAccount("Deployer");
```
#### Let dig into API provided by `locklift-deploy`
1. `locklift.deployments.createAccount` as the parameter it takes an array of create account config, all types of accounts supported
2. `locklift.deployments.deploy` takes an object with two fields `deployConfig` that equals `locklift.factory.deployContract` and `contractName` this is an identifier for getting access to the contract
3. `locklift.deployments.fixture(config?: { include?: Array<string>; exclude?: Array<string> })`
   this is a trigger for starting the deployment flow. This method takes the optional object as the parameter,
   it provides the possibility to control deployment flow e.g. exclude some scripts, or include some scripts. By default, all scripts will be run
4. `locklift.deployments.loadFromLogFile`_(migration)_ The `locklift-deploy` writes all deployed contracts and all created accounts to the log file.
   Which is inside the `deploy` folder and called `log.json`, so we can retrieve our state via this log file without redeploying anything
5. `locklift.deployments.clearLogFile` clear all deployments logs
