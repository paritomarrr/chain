# Hyperledger Fabric Chaincode in JavaScript: A Comprehensive Tutorial

## Prerequisites

1. **Basic Knowledge of Hyperledger Fabric**:  
   You should understand the fundamental concepts of Fabric, including:
   - The role of peers, orderers, and CAs.
   - The concept of channels.
   - The ledger structure (blockchain and state database).
   - Chaincode as the on-chain business logic.

2. **Familiarity with JavaScript/TypeScript**:  
   Chaincode can be written in Node.js using JavaScript or TypeScript. This guide uses JavaScript, but TypeScript can provide additional type safety if desired.

3. **Installed Software**:
   - **Node.js v14 or above**:  
     Ensure Node.js and npm are installed.  
     ```bash
     node -v
     npm -v
     ```
   - **Docker & Docker Compose**:  
     Required to run Fabric network components for development.
   - **Fabric Samples and Binaries**:  
     Download the Fabric samples repository, which includes test networks and scripts for setup.  
     ```bash
     git clone https://github.com/hyperledger/fabric-samples.git
     cd fabric-samples
     ```
     Follow the instructions in `README.md` to install the binaries and Docker images.

---

## Understanding the Chaincode Model

Chaincode in Hyperledger Fabric is essentially the business logic that runs on the peers. It can:

- Read and write data to the ledger’s state database.
- Implement complex logic for transactions.
- Enforce access control and ensure data integrity.

The chaincode API in Node.js typically involves extending a `Contract` class (provided by the `fabric-contract-api` library). Each public method represents a transaction function that can be invoked by network clients.

---

## Directory Structure for Chaincode

A common structure for chaincode written in JavaScript might look like this:

```
my-chaincode/
  ├── package.json
  ├── lib/
  │    └── myContract.js
  ├── index.js
  └── test/
       └── myContract.test.js
```

- **`index.js`**: The entry point that exports the contract classes to Fabric.
- **`lib/myContract.js`**: The main chaincode logic defined as a class that extends `Contract`.
- **`test/`**: Contains Mocha/Chai test scripts for unit testing the chaincode logic (optional but recommended).

---

## Writing Your First Chaincode in JavaScript

### Step 1: Initialize the Project

Create a new directory for your chaincode:

```bash
mkdir my-chaincode
cd my-chaincode
npm init -y
```

Install the required dependencies:

```bash
npm install fabric-contract-api fabric-shim
```

- **`fabric-contract-api`**: Provides the `Contract` class and decorators to define and structure chaincode logic.
- **`fabric-shim`**: Low-level chaincode interface, providing APIs for interacting with the ledger state and transaction context.

### Step 2: Implement the Contract

Create `index.js` in the project root:

```javascript
'use strict';

const MyContract = require('./lib/myContract');

module.exports.contracts = [MyContract];
```

Create `lib/myContract.js`:

```javascript
'use strict';

const { Contract } = require('fabric-contract-api');

class MyContract extends Contract {

    constructor() {
        super('MyContract'); 
    }

    async initLedger(ctx) {
        const initialAssets = [
            { assetID: 'asset1', color: 'blue', size: 5, owner: 'Tom', appraisedValue: 300 },
            { assetID: 'asset2', color: 'red', size: 3, owner: 'Jerry', appraisedValue: 200 }
        ];

        for (const asset of initialAssets) {
            await ctx.stub.putState(asset.assetID, Buffer.from(JSON.stringify(asset)));
            console.log(`Asset ${asset.assetID} initialized`);
        }
    }

    async createAsset(ctx, assetID, color, size, owner, appraisedValue) {
        const exists = await this.assetExists(ctx, assetID);
        if (exists) {
            throw new Error(`The asset ${assetID} already exists`);
        }

        const asset = {
            assetID,
            color,
            size: parseInt(size),
            owner,
            appraisedValue: parseInt(appraisedValue)
        };

        await ctx.stub.putState(assetID, Buffer.from(JSON.stringify(asset)));
        return asset;
    }

    async readAsset(ctx, assetID) {
        const assetBytes = await ctx.stub.getState(assetID);
        if (!assetBytes || assetBytes.length === 0) {
            throw new Error(`Asset ${assetID} does not exist`);
        }
        return JSON.parse(assetBytes.toString());
    }

    async updateAsset(ctx, assetID, color, size, owner, appraisedValue) {
        const asset = await this.readAsset(ctx, assetID);
        asset.color = color;
        asset.size = parseInt(size);
        asset.owner = owner;
        asset.appraisedValue = parseInt(appraisedValue);

        await ctx.stub.putState(assetID, Buffer.from(JSON.stringify(asset)));
        return asset;
    }

    async deleteAsset(ctx, assetID) {
        const exists = await this.assetExists(ctx, assetID);
        if (!exists) {
            throw new Error(`The asset ${assetID} does not exist`);
        }
        await ctx.stub.deleteState(assetID);
    }

    async assetExists(ctx, assetID) {
        const assetBytes = await ctx.stub.getState(assetID);
        return assetBytes && assetBytes.length > 0;
    }

    async transferAsset(ctx, assetID, newOwner) {
        const asset = await this.readAsset(ctx, assetID);
        asset.owner = newOwner;
        await ctx.stub.putState(assetID, Buffer.from(JSON.stringify(asset)));
        return asset;
    }
}

module.exports = MyContract;
```

---

## Running a Test Network and Deploying Chaincode

### Step 1: Use the Fabric Test Network

Hyperledger Fabric provides a sample test network located in `fabric-samples/test-network`. This is a great way to quickly stand up a basic network and deploy chaincode.

In a separate terminal, start the test network:

```bash
cd fabric-samples/test-network
./network.sh down
./network.sh up createChannel -c mychannel -ca
```

This sets up a network with:
- Two organizations (Org1 and Org2).
- A channel named `mychannel`.
- Certificate authorities for identity issuance.

### Step 2: Package the Chaincode

Go back to the `my-chaincode` directory and package the chaincode:

```bash
npm install
```

Package the chaincode into a tar.gz file:

```bash
peer lifecycle chaincode package mychaincode.tar.gz --path . --lang node --label mychaincode_1
```

---

## Interacting With the Deployed Chaincode

After deploying, use CLI commands to interact with your chaincode:

1. **Create an Asset**:
   ```bash
   peer chaincode invoke \
   --channelID mychannel \
   --name mychaincode \
   -c '{"Args":["MyContract:createAsset","asset3","green","10","Alice","500"]}'
   ```

2. **Read an Asset**:
   ```bash
   peer chaincode query \
   --channelID mychannel \
   --name mychaincode \
   -c '{"Args":["MyContract:readAsset","asset3"]}'
   ```

3. **Update an Asset**:
   ```bash
   peer chaincode invoke \
   --channelID mychannel \
   --name mychaincode \
   -c '{"Args":["MyContract:updateAsset","asset3","yellow","15","Bob","600"]}'
   ```

---

## Testing Chaincode Logic Locally

Set up unit tests for your chaincode using `fabric-shim` mock stubs. Install test libraries:

```bash
npm install mocha chai sinon --save-dev
```

Create a test file `test/myContract.test.js` and write tests using Mocha:

```javascript
'use strict';

const { ChaincodeMockStub } = require('fabric-shim');
const MyContract = require('../lib/myContract');
const { expect } = require('chai');

describe('MyContract', () => {
    let contract;
    let stub;

    beforeEach(() => {
        contract = new MyContract();
        stub = new ChaincodeMockStub('MyMockStub', contract);
    });

    it('should init the ledger', async () => {
        await stub.mockInit('tx1', ['MyContract:initLedger']);
        const asset1 = await stub.getState('asset1');
        expect(asset1).to.not.be.empty;
    });

    it('should create a new asset', async () => {
        await stub.mockInvoke('tx2', ['MyContract:createAsset','asset10','blue','7','Alice','250']);
        const asset = await stub.getState('asset10');
        const assetData = JSON.parse(asset.toString());
        expect(assetData.owner).to.equal('Alice');
    });
});
```

Run tests:

```bash
npx mocha --timeout 10000
```

---

## Conclusion

You’ve now learned how to:
- Set up a chaincode project in JavaScript.
- Implement CRUD operations in chaincode.
- Deploy and interact with the chaincode on a Fabric network.
- Write and execute unit tests.

This tutorial forms the foundation for building more complex and production-ready chaincode for Hyperledger Fabric.
