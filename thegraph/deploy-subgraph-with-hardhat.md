# Introduction

In this tutorial you will learn how to deploy a Solidity smart contract to the Ethereum Rinkeby testnet using HardHat, then create and deploy its subgraph to the Subgraph Studio.

![Subgraph Studio](https://github.com/figment-networks/datahub-learn/raw/master/assets/graph.png)

Topics covered in this tutorial:

- Deploying the smart contract to Rinkeby testnet using Hardhat
- Creating a subgraph in the Subgraph Studio
- Downloading the contract abi and then creating a subgraph from it
- Deploying the subgraph to Subgraph Studio

# Prerequisites

To successfully complete this tutorial, you will need to have a basic understanding of Ethereum, Solidity, and the NodeJS ecosystem.

You will also need some Rinkeby Ether in your Metamask wallet. Follow the instructions below:

- Login to your Twitter account at <https://twitter.com/>
- Tweet the following: `Requesting funds into <YOUR RINKEBY WALLET ADDRESS> on the #Rinkeby #Ethereum test network.`
- Go to <https://faucet.rinkeby.io/>, and paste the link to your tweet
- Click on the `Give me Ether` button
- You shall receive some Rinkeby Ether once your transaction is mined

# Requirements

- You will need Metamask installed in your browser. You can install it from <https://metamask.io/>
- You need to have a recent version of Node.js installed. We recommend using v14.17.6 LTS for compatibility.

# Project setup

Run these commands to install the yarn package manager and the graph CLI globally. These are required to build and deploy your subgraph.

```text
npm i -g yarn @graphprotocol/graph-cli
```

Then run the following commands to create a new directory called vendingMachine, create a new yarn package inside it, and then install HardHat as a dev-dependency of the project (meaning that it is a package that is only needed for local development and testing).

```text
mkdir vendingMachine
cd vendingMachine
yarn init --yes
yarn add --dev hardhat
```

Then run `npx hardhat` to initialize the HardHat project. This will output:

```text
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   8888b.  888888
888    888     "88b 888P"  d88" 888 888 "88b     "88b 888
888    888 .d888888 888    888  888 888  888 .d888888 888
888    888 888  888 888    Y88b 888 888  888 888  888 Y88b.
888    888 "Y888888 888     "Y88888 888  888 "Y888888  "Y888

Welcome to Hardhat v2.6.4

? What do you want to do? …
▸ Create a basic sample project
  Create an advanced sample project
  Create an empty hardhat.config.js
  Quit
```

Select "Create an empty hardhat.config.js" by scrolling down the menu with the arrow keys on your keyboard and press Enter. This will create a `hardhat.config.js` file in the root of the vendingMachine directory, with the following contents:

```javascript
/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.7.3",
};
```

The following command will install the HardHat plugins necessary to compile and deploy your contract:

```text
yarn add --dev @nomiclabs/hardhat-ethers ethers @nomiclabs/hardhat-waffle
```

# Writing the smart contract

Create a new directory called `contracts` inside the `vendingMachine` directory, then create a file inside the `contracts` directory called `VendingMachine.sol`.

Paste this Solidity code into that file:

```javascript
// SPDX-License-Identifier: MIT

pragma solidity 0.7.3;

contract VendingMachine {

    // Store the owner of this smart contract
    address owner;

    // A mapping is a key/value store. Here we store cupcake balance of this smart contract.
    mapping (address => uint) cupcakeBalances;

    // Events are necessary for The Graph to create entities
    event Refill(address owner, uint amount, uint remaining, uint timestamp, uint blockNumber);
    event Purchase(address buyer, uint amount, uint remaining, uint timestamp, uint blockNumber);

    // When 'VendingMachine' contract is deployed:
    // 1. set the deploying address as the owner of the contract
    // 2. set the deployed smart contract's cupcake balance to 100
    constructor() {
        owner = msg.sender;
        cupcakeBalances[address(this)] = 100;
    }

    // Allow the owner to increase the smart contract's cupcake balance
    function refill(uint amount) public onlyOwner {
        cupcakeBalances[address(this)] += amount;
        emit Refill(owner, amount, cupcakeBalances[address(this)], block.timestamp, block.number);
    }

    // Allow anyone to purchase cupcakes
    function purchase(uint amount) public payable {
        require(msg.value >= amount * 0.01 ether, "You must pay at least 0.01 ETH per cupcake");
        require(cupcakeBalances[address(this)] >= amount, "Not enough cupcakes in stock to complete this purchase");
        cupcakeBalances[address(this)] -= amount;
        cupcakeBalances[msg.sender] += amount;
        emit Purchase(msg.sender, amount, cupcakeBalances[address(this)], block.timestamp, block.number);
    }

    // Function modifiers are used to modify the behaviour of a function.
    // When "onlyOwner" modifier is added to a function, only the owner of this smart contract can execute that function.
    modifier onlyOwner {
        // Verifying that the owner is same as the one who called the function.
        require(msg.sender == owner, "Only owner callable");

        // This underscore character is called the "merge wildcard".
        // It will be replaced with the body of the function (to which we added the modifier),
        _;
    }
}

```

# Compiling the smart contract

To compile the smart contract via HardHat (which will still use the solc compiler), run:

```text
npx hardhat compile
```

Assuming there are no errors or warnings, this will output:

```text
Compiling 1 file with 0.7.3
Compilation finished successfully
```

The `VendingMachine.sol` smart contract has now been compiled successfully.

# Deploying the smart contract

To deploy to the Ethereum Rinkeby testnet with HardHat, you need to add a network entry to your `hardhat.config.js` file:

```javascript
require("@nomiclabs/hardhat-waffle");

// Replace this private key with your Rinkeby account private key
const RINKEBY_PRIVATE_KEY = "YOUR RINKEBY_PRIVATE_KEY";

module.exports = {
  solidity: "0.7.3",
  networks: {
    rinkeby: {
      url: `https://rinkeby-light.eth.linkpool.io/`,
      accounts: [`0x${RINKEBY_PRIVATE_KEY}`],
    },
  },
};
```

To get your `RINKEBY_PRIVATE_KEY`, open your browser and open Metamask. Make sure **Rinkeby Test Network** is selected. Go to Account details, and click on `Export Private Key`.

**Never ever share your private key with anyone!**

Create a directory called `scripts`, and paste the below code into a new file `deploy.js` inside scripts directory.

```javascript
async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying contracts with the account:", deployer.address);

  console.log("Account balance:", (await deployer.getBalance()).toString());

  const VendingMachine = await ethers.getContractFactory("VendingMachine");
  const vendingMachine = await VendingMachine.deploy();

  console.log("Contract address:", vendingMachine.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```


Finally, run:

```text
npx hardhat run scripts/deploy.js --network rinkeby
```

This will output:

```text
Deploying contracts with the account: <YOUR RINKEBY WALLET ADDRESS>
Account balance: <YOUR RINKEBY WALLET BALANCE>
Contract address: <YOUR RINKEBY SMART CONTRACT ADDRESS>
```

Yay! You just deployed your smart contract successfully to Rinkeby testnet!

We will need the smart contract address later.

# Creating the project in Subgraph Studio

First, you will want to head over to the Subgraph Studio at <https://thegraph.com/studio/>.

![Login to Subgraph Studio](https://github.com/figment-networks/datahub-learn/raw/master/assets/graph_connect.png)

Click on the **Connect Wallet** button. Choose a Metamask wallet to login with. Once you are authenticated, you shall see the below screen, where you can create your first subgraph.

![Create your first subgraph](https://github.com/figment-networks/datahub-learn/raw/master/assets/graph_create_subgraph.png)

Next, you need to give your subgraph a name. Give the name as **vending**. Once that's done, you will see this screen:

![Subgraph dashboard](https://github.com/figment-networks/datahub-learn/raw/master/assets/graph_subgraph_created.png)

On this screen, you can see details about the subgraph like your deploy key, the subgraph slug and status.

# Creating the subgraph

Run the following command to create the subgraph by downloading the contract ABI from the Rinkeby testnet. A new directory called `vending` will be created, and all node dependencies will be installed automatically.

```text
graph init --contract-name VendingMachine --index-events --studio --from-contract <YOUR RINKEBY SMART CONTRACT ADDRESS> --network rinkeby vending
```

Replace with your deployed VendingMachine smart contract address from above.

This will output:

```text
✔ Subgraph slug · vending
✔ Directory to create the subgraph in · vending
✔ Ethereum network · rinkeby
✔ Contract address · <YOUR RINKEBY SMART CONTRACT ADDRESS>
✔ Fetching ABI from Etherscan
✔ Contract Name · VendingMachine
———
  Generate subgraph from ABI
  Write subgraph to directory
✔ Create subgraph scaffold
✔ Initialize subgraph repository
✔ Install dependencies with yarn
✔ Generate ABI and schema types with yarn codegen

Subgraph vending created in vending

Next steps:

  1. Run `graph auth` to authenticate with your deploy key.

  2. Type `cd vending` to enter the subgraph.

  3. Run `yarn deploy` to deploy the subgraph.

Make sure to visit the documentation on https://thegraph.com/docs/ for further information.
```

It shall create three files:

- `subgraph.yaml`: This stores the [subgraph manifest](https://thegraph.academy/developers/working-with-the-graph/)
- `schema.graphql`: This defines the data to be stored and how it can be queried using GraphQL
- `src/mapping.ts`: This defines the mapping between blockchain events to the entities defined in the schema

# Deploying the subgraph

Before you can deploy your subgraph, you need to get your deploy key from <https://thegraph.com/studio/subgraph/vending/>.

Run the following command to set the deploy key for this yarn project.

```text
graph auth --studio <DEPLOY_KEY>
```

You should see the following output:

```text
Deploy key set for https://api.studio.thegraph.com/deploy/
```

Once that is done, we can run the following commands to deploy the subgraph to Subgraph Studio:

```text
cd vending
yarn deploy
```

You shall be prompted for a version label. You can choose `1.0.0`. You should see the following output:

```text
✔ Version Label (e.g. v0.0.1) · v1.0.0
  Skip migration: Bump mapping apiVersion from 0.0.1 to 0.0.2
  Skip migration: Bump mapping apiVersion from 0.0.2 to 0.0.3
  Skip migration: Bump mapping apiVersion from 0.0.3 to 0.0.4
  Skip migration: Bump mapping specVersion from 0.0.1 to 0.0.2
✔ Apply migrations
✔ Load subgraph from subgraph.yaml
  Compile data source: VendingMachine => build/VendingMachine/VendingMachine.wasm
✔ Compile subgraph
  Copy schema file build/schema.graphql
  Write subgraph file build/VendingMachine/abis/VendingMachine.json
  Write subgraph manifest build/subgraph.yaml
✔ Write compiled subgraph to build/
  Add file to IPFS build/schema.graphql
                .. QmWEbGspWEL95ipBPxDCFFrddswSBsprwx1Ktm35Xk2t5M
  Add file to IPFS build/VendingMachine/abis/VendingMachine.json
                .. QmPvqMksRmuchK4Q7kL2KpKg4ZZEsBXwHE34PE55YZUd1Y
  Add file to IPFS build/VendingMachine/VendingMachine.wasm
                .. QmYR3nJToYt9HeuLhXeZ4rw11JrNJyu1NE5n4LQ8DCYsyi
✔ Upload subgraph to IPFS

Build completed: QmZMbjNuaEm1KvAhVaqE29Hz4EJfdNcDwtHgmGifWNrutV

Deployed to https://thegraph.com/studio/subgraph/vending

Subgraph endpoints:
Queries (HTTP):     https://api.studio.thegraph.com/query/8676/vending/v1.0.0
Subscriptions (WS): https://api.studio.thegraph.com/query/8676/vending/v1.0.0
```

You have now deployed the subgraph to your Subgraph Studio account. It will now start the syncing process, where data is extracted from the historical blocks in Rinkeby Ethereum testnet by The Graph node. Depending on the amount of data, this process can take anywhere between a few minutes to a few hours. New blocks will be inspected by The Graph node as soon as they are mined.

Once the sync is complete, you can access the subgraph on the Playground to run your queries.

# Conclusion

Congratulations on finishing this tutorial! You have learned how to deploy a smart contract to the Rinkeby testnet using HardHat. You also learned how to create and deploy a subgraph for the smart contract on Subgraph Studio.

# About the Author

I'm Robin Thomas, a blockchain enthusiast with few years of experience working with various blockchain protocols. Feel free to connect with me on [GitHub](https://github.com/robin-thomas).
