# Introduction

In this tutorial, you will learn how to deploy & unit test smart contracts in solidity using Truffle. Before diving into unit testing we will implement a smart contract for testing. Grab a cup of tea, review the Smart-Contract and unit test them!

# Prerequisites

This tutorial assumes that you have a basic understanding of Solidity, Truffle, and blockchains.

# Requirements

- [Truffle](https://www.trufflesuite.com/)
- The [Solidity](https://docs.soliditylang.org/en/v0.8.9/) compiler (this is installed automatically by Truffle)
- [Metamask](https://metamask.io/)
- A DataHub account - Go to <https://datahub.figment.io/sign_up> and create an account

The complete code used in this tutorial is available [on Github](https://github.com/PowerStream3604/Polygon-unit-testing)

# Unit testing smart contracts with Truffle

**What is unit testing?**
Unit testing is a way of **testing a unit** - the smallest piece of code that can be logically isolated in a system. These units are mostly functions, subroutine, methods or properties.

Before we can test a smart contract, we will need to implement one.

The smart contract we'll implement is a Smart Contract that represents a **Corportion**. Similar to how corporations operate, our smart contract assigns **roles:**
**Owners :** Shareholders of the company who have limited access.
**Master :** The president/CEO of the company with full access.
**Admins :** Both **Owners** and **Master** together as a group.

**Features**:
1. **Master** has the right to **add** owner with **addOwner()** and **remove** owner with **removeOwner()**.
2. **Admins** (owner & master) have the right to transfer their share to anyone using the **giveShare()** function.
3. **Admins** (owner & master) have the right to transfer their share to one of the owners(not master) using the **addShare()** function.
4. **checkIfOwner()** returns whether the given address is owner or not.
5. **getMaster()** returns the address of the **master**.
6. **getOwners()** returns the list of **owner** addresses.
7. **Users** who are not in the boundary of **Admins** cannot transfer their share but can still **receive**.


**Events**:
1. **MasterSetup** is emitted when **Master** is set

```solidity
event MasterSetup(address indexed master);
```

2. **OwnerAddition** is emitted when **Owner** is added by **Master**

```solidity
event OwnerAddition(address indexed owner);
```

3. **OwnerRemoval** is emitted when **Owner** is removed by **Master**

```solidity
event OwnerRemoval(address indexed owner);
```

4. **Transfer** is emitted when share of admins gets transfer by either of these functions **addShare()**, **giveShare()**

```solidity
event Transfer(address indexed receiver, uint256 amount);
```

## Defining the smart contract

*Notes: We name this smart Contract as **Company** and use solidity version **0.8.7***

Before defining the events and functions, let's first define variables to be used in the smart contract.

```solidity
contract Company {
    
    // Company contract.
    // owners of the contract are the share holders of the company.
    
    address public master;
    // List to keep track of all owner's address
    address[] public owners;
    // Mapping to keep track of all owners of the company
    mapping(address => bool) isOwner;
    // Mapping to keep track of all balance of share holders of the company
    mapping(address => uint256) share;
    
}
```

After defining the variables, we should define events to be used in the Company Smart Contract.

```solidity
// Events
event MasterSetup(address indexed master);
event OwnerAddition(address indexed owner);
event Transfer(address indexed receiver, uint256 amount);
event OwnerRemoval(address indexed owner);
```

Since we defined the events, we should define modifier to limit access from **anonymous** or **unauthorized** users.

```solidity
modifier onlyMaster() {
    require(msg.sender == address(master));
    _;
}
modifier onlyOwners() {
    require(isOwner[msg.sender], "Only owners have the right to call");
    _;
}
modifier onlyAdmins() {
    require(isOwner[msg.sender] || msg.sender == address(master), "Only master or owners have the right to call");
    _;
}
```

We should define the constructor to set the address of the **Master**.

```solidity
/// @dev Constructor sets the master address of Company contract.
/// @param _master address to setup master 
constructor(address _master) {
    require(_master != address(0), "Master address cannot be a zero address");
    master = _master;
    share[master] = 10000000;
    emit MasterSetup(master);
}
```

Also, we should define functions to get the information about **Master** and **Owners**.

```solidity
/// @dev Returns the address of the master
function getMaster()
    public
    view
    returns (address)
{
    return master;
}

/// @dev Returns the owner list of this Company contract.
function getOwners()
    public
    view
    returns (address[] memory)
{
    return owners;
}
```

We would need a function to check if the given address is owner or not.

```solidity
/// @dev Returns whether the given address is owner or not.
/// @param owner Address to check if is owner.
function checkIfOwner(address owner)
    public
    view
    returns (bool)
{
    return isOwner[owner];
}
```

We would also need functions to **add** and **remove** owners.

```solidity
/// @dev Adds owner if the msg.sender is master. Will revert otherwise.
/// @param owner Owner address to be added as owner.
function addOwner(address owner) 
    onlyMaster 
    public
{
    isOwner[owner] = true;
    owners.push(owner);
    share[owner] = 5000000;
    
    emit OwnerAddition(owner);
}

/// @dev Removes an owner from the owner list. Can only be called by master. Will Revert otherwise
/// @param owner Address of the owner to be removed
function removeOwner(address owner)
    public
    onlyMaster
{
    require(isOwner[owner], "Only owners can be removed from owner list");
    isOwner[owner] = false;
    for (uint i = 0; i < owners.length - 1; i++)
        if (owners[i] == owner) {
            owners[i] = owners[owners.length - 1];
            break;
        }
    owners.pop();
        
    emit OwnerRemoval(owner);
}
```

The most important part of all, we need functions to transfer share.

```solidity
/// @dev Transfers owner's or master's share to any address given.
///     Note: can only be called by one of the owners or master
/// @param receiver Address of the receiver who'll receive the share
/// @param _share Uint256 of the amount the admin wants to transfer
function giveShare(address receiver, uint256 _share)
    public
    onlyAdmins
{
    require(share[msg.sender] >= _share, "Share exceeds the sender allowance");
    share[msg.sender] = share[msg.sender] - _share;
    share[receiver] += _share;
        
    emit Transfer(receiver, _share);
}
    
/// @dev Transfers owner's or master's stake(share) to an address in the owner list or master.
///     Note: the recipient can only be one of the admins(owner or master)
/// @param receiver Address of the receipient
/// @param _share Uint256 amount of the stake(share) to transfer
function addShare(address receiver, uint256 _share)
    public
    onlyAdmins
{
    require(share[msg.sender] >= _share, "Share exceeds the sender allowance");
    require(isOwner[receiver], "The receipient should only be one of the owners");
    share[msg.sender] -= _share;
    share[receiver] += _share;
        
    emit Transfer(receiver, _share);
}
```

Here is the full **implementation**:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

    
contract Company {
    
    // Company contract.
    // owners of the contract are the share holders of the company.

    
    address public master;
    // List to keep track of all owner's address
    address[] public owners;
    // Mapping to keep track of all owners of the company
    mapping(address => bool) isOwner;
    // Mapping to keep track of all balance of share holders of the company
    mapping(address => uint256) share;
    
    
    // Events
    event MasterSetup(address indexed master);
    event OwnerAddition(address indexed owner);
    event Transfer(address indexed receiver, uint256 amount);
    event OwnerRemoval(address indexed owner);
    
    
    modifier onlyMaster() {
        require(msg.sender == address(master));
        _;
    }
    modifier onlyOwners() {
        require(isOwner[msg.sender], "Only owner can be added");
        _;
    }
    modifier onlyAdmins() {
        require(isOwner[msg.sender] || msg.sender == address(master), "Only master or owners");
        _;
    }
    
    /// @dev Constructor sets the master address of Company contract.
    /// @param _master address to setup master 
    constructor(address _master) {
        require(_master != address(0), "Master address cannot be a zero address");
        master = _master;
        share[master] = 10000000;
        emit MasterSetup(master);
    }
    
    /// @dev Returns the address of the master
    function getMaster()
        public
        view
        returns (address)
    {
        return master;
    }
    
    /// @dev Adds owner if the msg.sender is master. Will revert otherwise.
    /// @param owner Owner address to be added as owner.
    function addOwner(address owner) 
        onlyMaster 
        public
    {
        isOwner[owner] = true;
        owners.push(owner);
        share[owner] = 5000000;
        
        emit OwnerAddition(owner);
    }
    
    /// @dev Returns the owner list of this Company contract.
    function getOwners()
        public
        view
        returns (address[] memory)
    {
        return owners;
    }
    
    /// @dev Returns whether the given address is owner or not.
    /// @param owner Address to check if is owner.
    function checkIfOwner(address owner)
        public
        view
        returns (bool)
    {
        return isOwner[owner];
    }

    /// @dev Transfers owner's or master's share to any address given.
    ///     Note: can only be called by one of the owners or master
    /// @param receiver Address of the receiver who'll receive the share
    /// @param _share Uint256 of the amount the admin wants to transfer
    function giveShare(address receiver, uint256 _share)
        public
        onlyAdmins
    {
        require(share[msg.sender] >= _share, "Stake exceeds the sender allowance");
        share[msg.sender] = share[msg.sender] - _share;
        share[receiver] += _share;
        
        emit Transfer(receiver, _share);
    }
    
    /// @dev Transfers owner's or master's stake(share) to an address in the owner list or master.
    ///     Note: the recipient can only be one of the admins(owner or master)
    /// @param receiver Address of the receipient
    /// @param _share Uint256 amount of the stake(share) to transfer
    function addShare(address receiver, uint256 _share)
        public
        onlyAdmins
    {
        require(share[msg.sender] >= _share, "Share exceeds the sender allowance");
        require(isOwner[receiver], "The receipient should only be one of the owners");
        share[msg.sender] = share[msg.sender] - _share;
        share[receiver] += _share;
        
        emit Transfer(receiver, _share);
    }
    
    /// @dev Removes an owner from the owner list. Can only be called by master. Will Revert otherwise
    /// @param owner Address of the owner to be removed
    function removeOwner(address owner)
        public
        onlyMaster
    {
        require(isOwner[owner], "Only owners can be removed from owner list");
        isOwner[owner] = false;
        for (uint i = 0; i < owners.length - 1; i++)
            if (owners[i] == owner) {
                owners[i] = owners[owners.length - 1];
                break;
            }
        owners.pop();
        
        emit OwnerRemoval(owner);
    }
}
```

**Hooray!!** We implemented the whole contract!!

Let's then go to unit test the above contract with Truffle.

As I mentioned above, unit testing is testing the **smallest** unit which can be functions, subroutines, methods, etc.
Truffle provides a convinient library to test smart contracts, by using **truffle-assert** library, we'll check if all scenarios stand by our expectation.

## Initialize Truffle project

To initialize a new Truffle project, run the command `truffle init` inside the directory you want to use, for example `polygon-unit-testing`. You can name the directory whatever you want, but we recommend using this example to follow the tutorial:

```text
mkdir polygon-unit-testing
cd polygon-unit-testing
truffle init
```

Then, you'll see a project directory like this:

![project overview](../../../.gitbook/assets/truffle-initial-directory.png)

## Paste your smart contract into the contracts folder

Create a new file named `Company.sol` file under `/contracts`

**Then**, paste the smart contract **implementation** from the previous section into `Company.sol`.

## Configure the network settings

**Edit** the Truffle configuration file, `truffle-config.js`.

Inside the **networks** object paste the network configuration shown below:

```javascript
mumbai: {
      provider: () => new HDWalletProvider(["<Private Key 1>", "<Private Key 2>", "<Private Key 3>"],
      `https://matic-mumbai--rpc.datahub.figment.io/apikey/${process.env.DATAHUB_POLYGON_API_KEY}`),
      network_id: 80001,
      confirmations: 2,
      timeoutBlocks: 200,
      skipDryRun: true,
      networkCheckTimeout: 100000,
}
```

This configuration sets the provider URL to connect Truffle with a DataHub node of the **Mumbai testnet**, and provides private keys to **sign** and pay for **gas fees** on **Mumbai**. Remember to replace <Private Key 1>, etc.. with the actual private keys you will be using.

**dotenv and .env**:

If you are unfamiliar with how to use `.env` files, refer to [this guide](https://docs.figment.io/network-documentation/extra-guides/dotenv-and-.env).

You must add your DataHub API key to a file named `.env` in the same directory as `truffle-config.js`, as the value of the environment variable `DATAHUB_POLYGON_API_KEY`:

```text
DATAHUB_POLYGON_API_KEY=<paste your API key here>
```

You will also need to add the code to use dotenv at the top of `truffle-config.js`:

```javascript
require('dotenv').config(); // Load .env file
```

You will need **3 distinct private keys** for this test. You can refer to this manual to export private keys from Metamask: <https://metamask.zendesk.com/hc/en-us/articles/360015289632-How-to-Export-an-Account-Private-Key>.

**NOTE**: The accounts should be funded with test **MATIC** on Mumbai. Use the [Polygon faucet](https://faucet.polygon.technology/).

## Create company.js file in test folder

In order to create tests in a Truffle project, create a test file under the `test` folder.

```text
cd test
touch company.js
```

## Create deployment file to deploy Company Contract

```text
cd migrations
touch 2_deploy_contract.js
```

To deploy a contract, we will use a migration script (migration is Truffle's way of saying deployment). 
Add the following code to `2_deploy_contract.js`:

```javascript
const Company = artifacts.require("Company");

module.exports = function (deployer, networks, accounts) {
    deployer.deploy(Company, account);
};
```

## Before unit testing smart contract

We'll use the JavaScript library of Truffle to test the functions. Before going in, we'll import the contract we want to test and the Truffle library for testing. Add this code to the test file:

```javascript
// polygon-unit-testing/test/company.js

const Company = artifacts.require("Company");
const truffleAssert = require('truffle-assertions');
```

Also, we'll define user variables to better distinguish accounts for testing we designated in the **truffle-config.js** file:

```javascript
// polygon-unit-testing/test/company.js

const user1 = accounts[0];
const user2 = accounts[1];
const user3 = accounts[2];
```

## Writing unit tests

1. Test if the Master address is set appropariately by the constructor.

```javascript
// polygon-unit-testing/test/company.js

it("1. should be able to set the right master", async () => {
    // Get deployed contract
    const company = await Company.deployed();
    // Check if the master address equals to user1
    assert.deepEqual(await company.getMaster(), user1);
});
```

2. Check if only master is able to add owner

```javascript
// polygon-unit-testing/test/company.js

it("2. only master should be able to add owner", async () => {
    // Get deployed contract
    const company = await Company.deployed();
    // Check if the user2 (is not master) gets reverted when attempting to add owner
    await truffleAssert.reverts(
        company.addOwner(user1, {from: user2}),
    );
});
```

3. Check if Master is able to add owner.

```javascript
// polygon-unit-testing/test/company.js

it("3. master should be able to add owner", async() => {
    // Get deployed contract
    const company = await Company.deployed();
    // call addOwner(); {from: accounts[0]} is added as default (who is master)
    const tx = await company.addOwner(user1);
    // Check if OwnerAddition is getting emitted
    truffleAssert.eventEmitted(tx, "OwnerAddition", (ev) => {
        return ev.owner == user1;
    })
    // Check if getOwners() returns the ownerList with user1 as owner inside
    assert.deepEqual(await company.getOwners(), [user1]);
});
```

4. Check if an address is an owner

```javascript
// polygon-unit-testing/test/company.js

it("4. should be able to check owner", async () => {
    const company = await Company.deployed();
    await company.addOwner(user1);

    // checkIfOwner() should true since user1 is in the ownerList   
    assert.equal(await company.checkIfOwner(user1), true);
});
```

5. Check if master is the only account to remove owner

```javascript
// polygon-unit-testing/test/company.js

it("5. only master should be able to remove owners", async () => {
    const company = await Company.deployed();

    await truffleAssert.reverts(
        company.removeOwner(user1, {from: user2}),
    );
});
```

6. Check if Master is able to add owner

```javascript
// polygon-unit-testing/test/company.js

it("6. master should be able to remove owners", async () => {
    const company = await Company.deployed();

    const tx = await company.removeOwner(user1);
    // check if OwnerRemoval event is emitted
    truffleAssert.eventEmitted(tx, "OwnerRemoval", (ev) => {
        return ev.owner == user1;
    });
    // ownerList should be empty
    assert.deepEqual(await company.getOwners(), []);
});
```

7. Check if Master is able to send his share to owners

```javascript
// polygon-unit-testing/test/company.js

it("7. master should be able to send his share to owners", async() => {
    const company = await Company.deployed();
    // add user2 as owner
    await company.addOwner(user2);

    // the initial share is 5000000
    assert.equal(await company.getShare(user2), 5000000);

    // add the share of master to user2
    const tx1 = await company.addShare(user2, 1000);

    // check if Transfer event is getting emitted
    truffleAssert.eventEmitted(tx1, "Transfer", (ev) => {
        return ev.receiver == user2 && ev.amount == 1000
    });
    // get the share of user2
    const user2Share = await company.getShare(user2);

    // user2 share should be 5000000 + 1000 : 5001000
    assert.equal(user2Share.toString(), 5001000);
});
```

8. Check if it's not possible to user `addShare()` to transfer share to ordinary users

```javascript
// polygon-unit-testing/test/company.js

it("8. should not be able to use addShare() to transfer share to non-admins(normal-users)", async() => {
    const company = await Company.deployed();

    await truffleAssert.reverts(
        company.addShare(user3, 1000),
    );
});
```

9. Check if it's possible to use `giveShare()` to transfer share to ordinary users

```javascript
// polygon-unit-testing/test/company.js

it("9. should be able to use giveShare() to transfer share to anyone", async () => {
    const company = await Company.deployed();

    const tx = await company.giveShare(user3,  1500);
    truffleAssert.eventEmitted(tx, "Transfer", (ev) => {
        return ev.receiver == user3 && ev.amount == 1500;
    });

    assert.equal(await company.getShare(user3), 1500);
});
```

The complete test file should look like this:

```javascript
// polygon-unit-testing/test/company.js

const Company = artifacts.require("Company");
const truffleAssert = require("truffle-assertions");
contract("Company", (accounts) => {
  const user1 = accounts[0];
  const user2 = accounts[1];
  const user3 = accounts[2];
  console.log(accounts);

  it("1. should be able to set the right master", async () => {
    const company = await Company.deployed();
    assert.deepEqual(await company.getMaster(), user1);
  });

  it("2. only master should be able to add owner", async () => {
    const company = await Company.deployed();
    await truffleAssert.reverts(company.addOwner(user1, { from: user2 }));
  });

  it("3. master should be able to add owner", async () => {
    const company = await Company.deployed();
    const tx = await company.addOwner(user1);
    truffleAssert.eventEmitted(tx, "OwnerAddition", (ev) => {
      return ev.owner == user1;
    });

    assert.deepEqual(await company.getOwners(), [user1]);
  });

  it("4. should be able to check owner", async () => {
    const company = await Company.deployed();
    await company.addOwner(user1);

    assert.equal(await company.checkIfOwner(user1), true);
  });

  it("5. only master should be able to remove owners", async () => {
    const company = await Company.deployed();

    await truffleAssert.reverts(company.removeOwner(user1, { from: user2 }));
  });

  it("6. master should be able to remove owners", async () => {
    const company = await Company.deployed();

    const tx = await company.removeOwner(user1);
    truffleAssert.eventEmitted(tx, "OwnerRemoval", (ev) => {
      return ev.owner == user1;
    });
    assert.deepEqual(await company.getOwners(), []);
  });

  it("7. master should be able to send his share to owners", async () => {
    const company = await Company.deployed();

    await company.addOwner(user2);

    assert.equal(await company.getShare(user2), 5000000);

    const tx1 = await company.addShare(user2, 1000);
    truffleAssert.eventEmitted(tx1, "Transfer", (ev) => {
      return ev.receiver == user2 && ev.amount == 1000;
    });

    const user2Share = await company.getShare(user2);

    assert.equal(user2Share.toString(), 5001000);
  });

  it("8. should not be able to use addShare() to transfer share to non-admins(normal-users)", async () => {
    const company = await Company.deployed();

    await truffleAssert.reverts(company.addShare(user3, 1000));
  });

  it("9. should be able to use giveShare() to transfer share to anyone", async () => {
    const company = await Company.deployed();

    const tx = await company.giveShare(user3, 1500);
    truffleAssert.eventEmitted(tx, "Transfer", (ev) => {
      return ev.receiver == user3 && ev.amount == 1500;
    });

    assert.equal(await company.getShare(user3), 1500);
  });
});
```

## Run the test

```text
truffle test --network mumbai
```

The result of the test should look similar: 

![Test Result](../../../.gitbook/assets/Polygon-unit-testing-result.png)


# Conclusion

Congratulations, you have finished this tutorial about unit testing Solidity smart contracts with Truffle. After completing this tutorial you'll able to write a basic smart contract with Solidity and use the Truffle library to unit test it.

# About the Author

David Kim is a Blockchain developer interested in NFTs and DeFi. Contact him on [GitHub](https://github.com/PowerStream3604).

# References

- [Truffle](https://www.trufflesuite.com/)
- [Solidity](https://docs.soliditylang.org/en/v0.8.9/)
- [Polygon docs](https://docs.matic.network/docs/develop/getting-started)
