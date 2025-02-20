# Introduction
In this tutorial, we will discuss how to create an NFT marketplace on the Tezos blockchain. We will build a simple open NFT marketplace where anyone can mint and collect NFTs. We will be going through writing the smart contracts,  creating the UI, and invoking the entry points in the front end.

# Prerequisites
- Basic familiarity with [ReactJS](https://reactjs.org/) and Redux.
- Should also have some experience writing smart contracts on Tezos with SmartPy.

# Requirements
- [Git](https://git-scm.com/downloads)
- A wallet installed in your browser for Testing. We will use [Temple wallet](https://templewallet.com/)
- Node.js
- npm

# Introduction to NFTs and IPFS
## What are NFTs?
A non-fungible token is a unique and non-interchangeable unit of data stored on a digital ledger. NFTs can be used to represent easily-reproducible items such as photos, videos, audio, and other types of digital files as unique items, and use blockchain technology to establish a verified and public proof of ownership.

## What is IPFS?
The InterPlanetary File System is a protocol and peer-to-peer network for storing and sharing data in a distributed file system. IPFS uses content-addressing to uniquely identify each file in a global namespace connecting all computing devices. In this tutorial, we will be using [nft.storage](nft.storage) to store the metadata for NFTs.

# Smart Contract
We will be building two contracts for the marketplace. First will be the token contract. On Tezos [FA2](https://tezos.b9lab.com/fa2) is the standard for Non-fungible Token contracts. We will be using the template provided by SmartPy to build out the Token Contract.
The template contains the basic entry points for building a Fungible or Non-fungible token including
- Minting tokens
- Transfer
- Adding operators

You can check out the template for yourself [on the SmartPy site](https://smartpy.io/dev/templates/FA2.py).



## Token contract
We will use SmartPy online IDE to build out the contract. You can also use the SmartPy CLI if you prefer.

1. Go to [smartpy.io/ide](smartpy.io/ide)
2. We will first import the Smartpy library

```python
import smartpy as sp
```

3. Now, we can directly import the FA2 template from smartpy with the `import_script_from_url` function

```python
FA2 = sp.io.import_script_from_url("https://smartpy.io/dev/templates/FA2.py")
```
4. FA2 template is a smart contract ready to be deployed for Fungible and Non-fungible tokens.
5. To use the imported contract, we can create a class and inherit it from the FA2 template.

```python
class Token(FA2.FA2):
    pass
```

6. That is all for the NFT contract. Now, we will write some tests to see if this works. Create tests by using `@sp.add_test` decorator from the SmartPy library.

```python
@sp.add_test(name = "Non Fungible Token")
def test():
```

7. First, we start by creating a scenario and some test accounts.

```python
@sp.add_test(name = "Non Fungible Token")
def test():
    scenario = sp.test_scenario()
    
    admin = sp.test_account("admin")
    mark = sp.test_account("user1")
    elon = sp.test_account("user2")
```

8. Now, we create the instance of the Token contract and pass the required parameters which are:

- FA2 config
- Admin address
- Metadata

You can customize all the parameters however you like and change the metadata IPFS hash. Here we are sticking with a simple configuration.

```python
@sp.add_test(name = "Non Fungible Token")
def test():
    # ...
    token_contract = Token(FA2.FA2_config(non_fungible = True), admin = admin.address, metadata = sp.utils.metadata_of_url("ipfs://QmW8jPMdBmFvsSEoLWPPhaozN6jGQFxxkwuMLtVFqEy6Fb"))
    
    scenario += token_contract
```
> Note: Where ever there is missing code in the tutorial ellipsis is used to indicate it.

9. We are all set for the Token contract. We can test this out by clicking on the run button.

## Marketplace contract
A marketplace contract handles minting and managing the NFTs. We usually set this contract as admin of the NFT contract. Then we use inter-contract calling to mint, transfer, collect NFTs.
There are some important features for a marketplace.
- Minting NFT
- Collecting NFT
- Transfering XTZ from contract to account. i.e. Collecting management rewards
- Updating admin

Before writing the functions, let us create the contract boilerplate. We will name our class `MarketPlace`. Let's initialize data fields in the init function. 
We are going to initialize 
- `token`: The address to the token contract.
- `metadata`: Meta-data of this contract to make it recognizable in explorers.
- `admin`: the admin for this contract,
- `token_id`: we will use this to know the total number of NFTs we have minted.
- `data`: This is a big map to save a particular token's data.
	- `holder`: The current owner of this NFT.
	- `collectable`: Whether the NFT is on sale or not. 
	- `author`: The creator of the NFT.
	- `amount`: The amount at which this NFT will be sold, if the item is collectible

```python
class Marketplace(sp.Contract):
    def __init__(self, token, metadata, admin):
        self.init(
            token = token,
            metadata = metadata,
            admin = admin,
            data = sp.big_map(tkey=sp.TNat, tvalue=sp.TRecord(holder=sp.TAddress, author = sp.TAddress, amount=sp.TNat, token_id=sp.TNat, collectable=sp.TBool)),
            token_id = 0,
            )
```

Now, Let us code the entry points one by one.

## Minting NFT
The mint function is used for minting new NFTs. We first verify the amount at which the minter wants to sell this NFT is positive.
Then we call our token contract, by inter-contract calling. Once the inter contract call is made, we update our `data` big map. Here, We are also sending metadata to the `Token` contract.

We add a new token in our big map and set the holder as the contract, author as the sender, amount at which this will be sold as the input parameter `amount` and `collectable` True.

```python
    @sp.entry_point
    def mint(self, params):
        sp.verify((params.amount > 0))
        c = sp.contract(
            sp.TRecord(
            address=sp.TAddress,
            amount=sp.TNat,
            token_id=sp.TNat,
            metadata=sp.TMap(sp.TString, sp.TBytes)
            ), 
            self.data.token, 
            entry_point = "mint").open_some()
            
        sp.transfer(
            sp.record(
            address = sp.self_address,
            amount = 1,
            token_id = self.data.token_id,
            metadata={ '' : params.metadata }
            ), 
            sp.mutez(0), 
            c)
        
        self.data.data[self.data.token_id] = sp.record(holder=sp.self_address, author = sp.sender, amount = params.amount, token_id=self.data.token_id, collectable=True)
        self.data.token_id += 1
```

## Collect NFT entry point
Before we start coding this, let us first create a utility function that we can use to call the transfer entry point in the token contract.

```python
    def fa2_transfer(self, fa2, from_, to_, token_id, amount):
            c = sp.contract(sp.TList(sp.TRecord(from_=sp.TAddress, txs=sp.TList(sp.TRecord(amount=sp.TNat, to_=sp.TAddress, token_id=sp.TNat).layout(("to_", ("token_id", "amount")))))), fa2, entry_point='transfer').open_some()
            sp.transfer(sp.list([sp.record(from_=from_, txs=sp.list([sp.record(amount=amount, to_=to_, token_id=token_id)]))]), sp.mutez(0), c)
```

We pass the contract address and the information about the token transfer.

Now, we can write our entry point.

```python
   @sp.entry_point
    def collect(self, params):
        sp.verify(((sp.amount == sp.utils.nat_to_mutez(self.data.data[params.token_id].amount)) & (self.data.data[params.token_id].amount != 0) & (self.data.data[params.token_id].collectable == True) & (self.data.data[params.token_id].author != sp.sender)))
        self.data.data[params.token_id].collectable = False
        self.data.data[params.token_id].holder = sp.sender

        #sending rewards
        sp.send(self.data.data[params.token_id].author, sp.split_tokens(sp.amount, 97, 100))
        
        self.fa2_transfer(self.data.token, sp.self_address, sp.sender, params.token_id, 1)
```

Here, we will first verify that amount with the transaction is equal to the cost of the NFT.

> Note: The amount of mutez for the NFT is stored in the `amount` key in our `data` big map.

Then we will also verify if the NFT was on sale. That is `collectable` key is True.

If all the checks pass we can set our new holder. And we will use our utility function to call the Token smart contract and transfer the NFT.

We are also sending 97% of the amount to the author of the NFT.

## Update admin Entry point
We check if the sender is admin and, if it is true we will update the admin.

> It is always good to have an entry point to update admin, in case you have lost your private key.

```python
    @sp.entry_point
    def update_admin(self, params):
        sp.verify(sp.sender == self.data.admin)
        self.data.admin = params
```

## Collect management rewards
Our smart contract has collected XTZ and we want to withdraw them. We are creating an entry point just for that.
We verify the sender to be admin and transfer the amount of XTZ, specified in parameters.

```python
    @sp.entry_point
    def collect_management_rewards(self, params):
        sp.verify(sp.sender == self.data.admin)

        sp.send(params.address, params.amount)
```

Hurray! We have coded all the entrypoints now. Next, we can write tests for all of them.

## Testing the contracts.
First, we initialize the `MarketPlace` Contract.
We will pass the token address as the first parameter. We get the token address by calling `.address` on the Token contract instance.

```python
@sp.add_test(name = "Non Fungible Token")
def test():
    scenario = sp.test_scenario()
    
    admin = sp.test_account("admin")
    mark = sp.test_account("user1")
    elon = sp.test_account("user2")
    
    
    token_contract = Token(FA2.FA2_config(non_fungible = True), admin = admin.address, metadata = sp.utils.metadata_of_url("ipfs://QmW8jPMdBmFvsSEoLWPPhaozN6jGQFxxkwuMLtVFqEy6Fb"))
    
    scenario += token_contract

    scenario.h1("MarketPlace")
    marketplace = Marketplace(token_contract.address, sp.utils.metadata_of_url("ipfs://QmW8jPMdBmFvsSEoLWPPhaozN6jGQFxxkwuMLtVFqEy6Fb"), admin.address)
    scenario += marketplace
```

Now we will try to mint the NFT, we use a temporary IPFS hash here. First, we will call the entry point without giving the `MarketPlace` contract admin privileges. This is to check if everything is working how it is supposed to. We should see a fail message if we try to run.
After that, we can call the `set_administrator` entry point on the Token contract and set the Marketplace contract as the admin.
If we try to mint again, it will work.

Now we can test out the collect function.
> Make sure to use a different user while calling the entry point. As we have added a check to make sure the minter is not the same as the collector.

Now, we can also try out the `collect_management_rewards` entry point and enter the amount and address in parameters.

```python
    scenario.h1("Mint")
    scenario += marketplace.mint(sp.record(amount = 100000000, metadata = sp.pack("ipfs://bafyreibwl5hhjgrat5l7cmjlv6ppwghm6ijygpz2xor2r6incfcxnl7y3e/metadata.json"))).run(sender = admin, valid = False)
    scenario += token_contract.set_administrator(marketplace.address).run(sender = admin)
    scenario += marketplace.mint(sp.record(amount = 100000000, metadata = sp.pack("ipfs://bafyreibwl5hhjgrat5l7cmjlv6ppwghm6ijygpz2xor2r6incfcxnl7y3e/metadata.json"))).run(sender = admin)
    scenario += marketplace.mint(sp.record(amount = 5600000, metadata = sp.pack("123423"))).run(sender = mark)
    scenario.h1("Collect")
    scenario += marketplace.collect(sp.record(token_id = 1)).run(sender = elon, amount = sp.mutez(5600000))

    scenario += marketplace.collect_management_rewards(sp.record(amount = sp.mutez(1000), address = admin.address)).run(sender = admin)
```

Check out the complete code on SmartPy [here](https://smartpy.io/ide?cid=QmVGqtBwNNnxwdFPrnmZYnQGq5PEQGSGMQwrB53rMtuJfg&k=6c6a8bdfdaf5a2893854).


## Deploy the contract

Now, we are going to deploy both contracts.

In the SmartPy output, click on `Deploy Michelson Project`

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-10.png)

Then will open a new page. Select the network where you want to deploy the smart contract.
Then select the account which will be used to deploy the smart contract.

Now Click on the `Estimated Cost From Rpc` to estimate the fee in Tezos to deploy the contract. Make sure the account you are using has that much XTZ available.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-9.png)


Before deploying we will change the admin for this contract. Update the address string to your public address.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-3.png)


Now click the `Deploy contract` button. This will generate the address for the deployed contract; save that.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-7.png)

And we have deployed the `Token` contract successfully. Now let us deploy the `MarketPlace` Contract. 


Now again, In the Smartpy output, click the deploy contract button for the `MarketPlace` Contract.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-5.png)

We will repeat the same steps for this one also. First, we will select the account, then we will estimate the cost.

Once these 2 steps are done, we will update the initial storage. Change the contract address here to the contract address we got after deploying the `Token` contract.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-2.png)

And now we can deploy this contract as well by clicking the `Deploy contract` button, then save the address of this contract.

Now we have deployed both the contracts. Now we can update the `admin` by calling the `setAdminstator` entry point.

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/tezos-nft-market-1.png)

> Note: you can update the IPFS hash, which can be done in the tests.

# Front End

We will build the frontend using React.js, Redux for state management, Taquito for interacting with the contract, and Beacon SDK for wallet connection.

To get the boilerplate setup, we will clone a Template that includes a wallet connection and gives a basic idea of invoking the entry points.

## Setup
1. Go to https://github.com/NishantChandla/taquito-react-redux-template
2. Click the `Use this template` button. This will create a new repository for you.
3. Now, we can clone the created repository (replace URL with the actual URL of the repository you just created):

```text
git clone URL
```

4. Open up the cloned folder in VSCode, and we can get a quick look at the boilerplate and start coding.
5. First run the `npm install` command in your terminal. Then run `npm start`. This will start a development server for the project.

> We will write the contract calls in `src/actions/index.js`.

## Editing the template
1. We will edit the contract addresses in the `config/index.js` file. We will use the addresses we got after deploying.

```js
const config = {
        contractAddress : "KT1AmPpGN9g1ew29xYDRSyD7FCVSJ44xizye", 
        tokenAddress: "KT1TnBDd6YmYVUysUCq1CRb85B4yPhE2W7qR"
}

export default config;
```

Here, I have used the contract address I got after deploying. You have to replace them with your addresses.
`contractAddress`: Marketplace address
`tokenAddress`: FA2 contract address

2. We will have to change the test network as we have deployed our contract on GRANADANET and the template has FLORENCENET.
There are a couple of instances where we will change the network RPC URL.
- App.js

```js
    const [Tezos, setTezos] = useState(
        new TezosToolkit("https://granadanet.smartpy.io/")
    );
    // ...
    const wallet_instance = new BeaconWallet({
    name: "NFT marketplace",
    preferredNetwork: NetworkType.GRANADANET,
    // ...
```

- actions.js

```js
    // ...
    await wallet.requestPermissions({
        network: {
            type: NetworkType.GRANADANET,
            rpcUrl: "https://granadanet.smartpy.io/",
        },
    });
    // ...
    setTezos(new TezosToolkit("https://granadanet.smartpy.io/"));

```

We have to change all the URLs from FLORENCENET to GRANADANET.
> We are using a SmartPy node here. You can use whatever node you prefer.

3. You can change the name of the project in `package.json` and `public/index.html`

## Writing Contract calls
We will first create actions creators for contract calls in `actions/index.js`
- mintNFT
- collectNFT

## mintNFT
We use the Tezos toolkit instance to create an instance of our marketplace contract. Then we can use the instance to call the entry points on our contract. Mint entry point requires two parameters amount and metadata URL. We can create these are parameters for the action creator and pass them up from a form. Here we cannot send a string as the second parameter as the metadata should of bytes type, so we convert it to hex code (bytes).

```js
export const mintNFT = ({ Tezos, amount, metadata }) => {
	return async (dispatch) => {
		try {
			const contract = await Tezos.wallet.at(config.contractAddress);
			let bytes = "";
			for (var i = 0; i < metadata.length; i++) {
				bytes += metadata.charCodeAt(i).toString(16).slice(-4);
			}
			const op = await contract.methods.mint(amount, bytes).send();
			await op.confirmation();
            dispatch(fetchData());
		} catch (e) {
			console.log(e);
		}
	};
};
```

> We have added a dispatch to fetch data. We will create that action creator later.
Our mintNFT action creator is complete.

## collectNFT
As before, we use the Tezos toolkit instance to create an instance of our marketplace contract. Then we can use the instance to call the collect entry point on our contract. Collect entry point requires one parameter which is the token id. Additionally, we want to send the amount required to collect the NFT. We can do so by passing an object when calling the entry point.
> Here we have used mutez units.

```js
export const collectNFT = ({ Tezos, amount, id }) => {
	return async (dispatch) => {
		try {
			const contract = await Tezos.wallet.at(config.contractAddress);

			const op = await contract.methods
				.collect(id)
				.send({ mutez: true, amount: amount });
			await op.confirmation();
            dispatch(fetchData());
		} catch (e) {
			console.log(e);
		}
	};
};
```

We have added a dispatch to fetch data. We will create that action creator later. When we dispatch this action creator it will call our contract and buy the NFT specified.
Note that the user address is already sent when we call `.send()` on the entry point

## Getting the contract data
Here we have two options either we can read the contract storage directly using taquito or we can use an indexer API to get the data.
We will be using tzkt API to get the storage. There are a couple of reasons to do so. Getting the storage from the contract using taquito is slow and we will be getting the data multiple times in our app.

Check out the API specification at https://api.granadanet.tzkt.io/

We will need to install Axios to make the API calls.
Go to the terminal and run `npm install axios`

Also, we will need two utility functions to convert bytes into strings back. As you might remember we converted string to bytes when minting.

```js
export const hex2buf = (hex) => {
	return new Uint8Array(
		hex.match(/[\da-f]{2}/gi).map((h) => parseInt(h, 16))
	);
};

export function bytes2Char(hex) {
	return Buffer.from(hex2buf(hex)).toString("utf8");
}
```

We will create an action creator fetch data, and then we will call the API with our contract addresses, as we want data from both the token contract and the marketplace contract. We will call both of them and then combine them with data. The token contract returns an array with values `token_id` and `token_info`, containing bytes for IPFS hash. We convert bytes to a string using the `bytes2Char` function, then create a `get` request with Axios using the `https://ipfs.io/ipfs/` public node.
We then combine both of the results in an object and append it to an array `tokenData`.

```js
export const fetchData = () => {
	return async (dispatch) => {
		try {
			const response = await axios.get(
				`https://api.granadanet.tzkt.io/v1/contracts/${config.contractAddress}/bigmaps/data/keys`
			);
			const response1 = await axios.get(
				`https://api.granadanet.tzkt.io/v1/contracts/${config.tokenAddress}/bigmaps/token_metadata/keys`
			);
			const d1 = response.data;
			const d2 = response1.data;
			let tokenData = [];
			for (let i = 0; i < d1.length; i++) {
				const s = bytes2Char(d2[i].value.token_info[""])
					.split("//")
					.at(-1);

				const res = await axios.get("https://ipfs.io/ipfs/" + s);

				const l1 = d1[i].value;
				const l2 = res.data;
				tokenData[i] = {
					...l1,
					...l2,
					token_id: d2[i].value.token_id,
				};
			}
			console.log(tokenData);
			dispatch({ type: "SET_TOKEN_DATA", payload: tokenData });

		} catch (e) {
			console.log(e);
		}
	};
};
```

Here we dispatch the data with type `SET_TOKEN_DATA`. Now we will create reducers to store the data.

## Creating reducers
As we can see we already have a reducer created for storing the wallet configuration i.e. user address and user balance.
Now we will create a tokenDataReducer and store the data for our NFTs.

```js
const tokenDataReducer=(state=[], action)=>{
    switch(action.type){
        case "SET_TOKEN_DATA":
            return action.payload;
        default:
            return state;
    }
}
```

This is straight forward reducer that stores whatever data is dispatched with the `SET_TOKEN_DATA` type.

We will need to add this reducer to the combine reducers function call.

```js
const reducers = combineReducers({walletConfig: connectWalletReducer, tokenData: tokenDataReducer});
```

We are all set with fetching the data. 
Now we can move on to minting our first NFT.

At this point, you can go ahead and build your design that shows the NFTs. You can use the collect and mint action creators to interact with the contract. 

## Creating the UI
In this tutorial, we will be going over a basic frontend using `Semantic UI`

First, we will set up React router. Go to your terminal and install `react-router-dom`.

```text
npm install react-router-dom
```

Now we can use the `BrowserRouter` component from `react-router-dom`. React router requires us to keep this as a parent to the App component. First, we import the component, then we will use it.

```js

import { BrowserRouter } from "react-router-dom";

ReactDOM.render(
	<Provider store={store}>
		<PersistGate loading={null} persistor={persistStore(store)}>
			<BrowserRouter>
				<App />
			</BrowserRouter>
		</PersistGate>
	</Provider>,
	document.querySelector("#root")
);

```

Now we will create the necessary routes for our app. We will create three routes for the home, Creating NFT, Showing a detailed view of NFT.

In our App.js file, we will first import the `Route` and `Switch` components. Here we will also import Home, Create, Show components that we have not created yet. We will be creating them later.
We can also import `fetchData` action creator as we want to call all NFT data when the app initially starts and the App component is the best place to do so.

```js
import { fetchData, _walletConfig } from '../actions';
import { Route, Switch } from 'react-router';
import Home from './layouts/Home';
import Create from './layouts/Create';
import Show from './layouts/Show';
```

Now we can use the Switch and Route components to do the routing. Route component takes a path prop, which will be appended to the URL.

```jsx
    return (
        <div className="ui container">
            <Header Tezos={Tezos} setTezos={setTezos} wallet={wallet} />
            <div className="ui container">
                <Switch>
                    <Route path="/create">
                        <Create Tezos={Tezos}/>
                    </Route>
                    <Route path="/show/:id">
                        <Show Tezos={Tezos}/>
                    </Route>
                    <Route path="/">
                        <Home Tezos={Tezos}/>
                    </Route>
                </Switch>
            </div>
        </div>
    );
```

We also dispatch the `fetchData` action creator using the dispatch object created already.

```js
   useEffect(()=>{
        dispatch(fetchData());
    },[dispatch]);
```

Now we will change the routes in our `Header.js`. We will use the `Link` component from `react-router-dom`.
We can also make the files a little more organized and move `Header.js` to `component/sections`.

```js
import { Link } from 'react-router-dom';

// ...
   return (
            <div className="ui menu black" style={{'marginTop':'5px'}}>
                <a href="/#" className="ui header item">NFTs</a>
                <Link className="item" to="/">Home</Link>

                {selector.userAddress!==""?
                <Link className="item" to="/create">Create NFT</Link>
                :null}

                <div className="right menu">
                    {(selector.userAddress==="")?
                    <a href="/#" className="item" onClick={onClick}>Connect Wallet</a>:
                    <a href="/#" className="item" onClick={onClick}>Disconnect Wallet</a>}
                </div>
            </div>
        );
```

If you run `npm start` now you will get an error as the components don't exist yet. We will be fixing the errors now.

## Fixing the errors

We need to create three files `Create.js`, `Show.js`, `Home.js` in `components/layouts`. We will create a basic React component in each of the files to get rid of the error.

```js
const Create = () => {
    return (<div>Create</div>);
}
export default Create;
```
> Note: Remember to use the component name in each file, such as: const Show = () => {} and const Home = () => {}

## Creating a form to mint NFTs
Our mintNFT action creator requires two parameters amount and metadata IPFS hash. In this tutorial, we will use the `nft.storage` library to upload our NFT data to IPFS storage.

The FA2 standard requires us to have some necessary fields in the metadata which are `name`, `decimals`, and `symbol`. Here we will also be adding a description and the image hash. To get the image we will use the `use-file-picker` package.
Let us install both of them.

```text
npm install use-file-picker nft.storage
```

We will need to get an API key by signing up for an account on [nft.storage](nft.storage). You can go ahead and get the API key once you've created an account and logged in.

```js
import { useFilePicker } from "use-file-picker";
import { NFTStorage, File } from "nft.storage";

import { mintNFT } from "../../actions";
import { useDispatch } from "react-redux";

const apiKey = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkaWQ6ZXRocjoweDY0NmFmRGIyMjhhMGY1RjFhMURDNDQyMjFCQ0E4YTIwNTNlNWUzQzIiLCJpc3MiOiJuZnQtc3RvcmFnZSIsImlhdCI6MTYyNDY0NjcyNjUwMiwibmFtZSI6IlRleiBCeXRlcyJ9.JfjY624-uOXE9naYfN7Z8QfY1bbO_bZcVihhHL5ke3I";
const client = new NFTStorage({ token: apiKey });
// ...
```

First, we import all the required packages and set up the nft.storage client.

```js
const Create = ({ Tezos }) => {
	const dispatch = useDispatch();
	const [openFileSelector, { filesContent }] = useFilePicker({
		accept: [".png", ".jpg", ".jpeg"],
		multiple: false,
		readAs: "ArrayBuffer",
	});
	const [name, setName] = useState("");
	const [description, setDescription] = useState("");
	const [symbol, setSymbol] = useState("");
	const [amount, setAmount] = useState("0");
	const [error, setError] = useState("");
	const [loadingSubmit, setLoading] = useState(false);
```

Here we will declare all the states we need for our form. We also declare `openFileSelector` and `filesContent` from the `useFilePicker` hook.
You can also store additional details on the NFT.

Now we will create a submit button that simply calls the nft.storage client and generates the IPFS hash. Then we dispatch the mintNFT action creator.

The form here has labels and fields, it also has some basic validation.
Note CSS comes from semantic UI. The link tag is already present in the `public/index.html` file.

```js
	const onSubmit = (e) => {
		e.preventDefault();
		if (
			name === "" ||
			description === "" ||
			symbol === "" ||
			amount === "" ||
			!/^-?\d+$/.test(amount) ||
			filesContent.length === 0
		) {
			setError("Some Error Occurred. Please check entered details.");
			return;
		}
		setLoading(true);
		setError("");

		(async () => {
			const metadata = await client.store({
				name: name,
				description: description,
                decimals: 0,
                symbol: symbol,
				image: new File(
					[filesContent[0].content],
					filesContent[0].name,
					{ type: "image/" + filesContent[0].name.split(".")[1] }
				),
			});
            console.log(metadata);
            dispatch(mintNFT({Tezos, amount, metadata: metadata.url}))

			setLoading(false);
			setName("");
			setAmount("0");
			setDescription("");
			setSymbol("");
		})();
	};

	return (
		<div>
			<form className="ui form error">
				<div
					className={`field required ${
						loadingSubmit ? "disabled" : ""
					}`}
				>
					<label>Token Name</label>
					<input
						type="text"
						value={name}
						onChange={(e) => setName(e.target.value)}
						placeholder="Tez Bytes"
					/>
				</div>
				{name.length > 30 ? (
					<div className="ui error message">
						<div className="header">Too long!</div>
						<p>The name must be less than 30 letters.</p>
					</div>
				) : null}
				<div
					className={`field required ${
						loadingSubmit ? "disabled" : ""
					}`}
				>
					<label>Description</label>
					<input
						type="text"
						value={description}
						onChange={(e) => setDescription(e.target.value)}
						placeholder="A digital art piece!"
					/>
				</div>
				{description.length > 300 ? (
					<div className="ui error message">
						<div className="header">Too long!</div>
						<p>The Description must be less than 300 letters.</p>
					</div>
				) : null}
				<div
					className={`field required ${
						loadingSubmit ? "disabled" : ""
					}`}
				>
					<label>Symbol</label>
					<input
						type="text"
						value={symbol}
						onChange={(e) => setSymbol(e.target.value)}
						placeholder="TBY"
					/>
				</div>
				{Symbol.length > 10 ? (
					<div className="ui error message">
						<div className="header">Too long!</div>
						<p>The Symbol must be less than 10 letters.</p>
					</div>
				) : null}
				<div
					className={`field required ${
						loadingSubmit ? "disabled" : ""
					}`}
				>
					<label>
						Selling Amount (Mutez) (There is a 3% service fee)
					</label>
					<input
						type="text"
						value={amount}
						onChange={(e) => setAmount(e.target.value)}
						placeholder="Amount"
					/>
				</div>
				{!/^-?\d+$/.test(amount) && amount !== "" ? (
					<div className="ui error message">
						<div className="header">Only number allowed</div>
						<p>The amount must be a valid Mutez value.</p>
					</div>
				) : null}
				<div
					className={`field required ${
						loadingSubmit ? "disabled" : ""
					}`}
				>
					<label>Image</label>
					<button
						type="button"
						className="ui basic button"
						onClick={(event) => {
							openFileSelector();
							event.preventDefault();
						}}
					>
						Select files{" "}
					</button>
					{filesContent.length > 0 ? filesContent[0].name : ""}
				</div>
				{error ? (
					<div className="ui error message">
						<div className="header">Error</div>
						<p>{error}</p>
					</div>
				) : null}

				<button
					className={`ui button ${loadingSubmit ? "loading" : ""}`}
					onClick={(e) => onSubmit(e)}
					type="submit"
				>
					Mint
				</button>
			</form>
		</div>
	);
};

export default Create;

```

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/Tezos-nft-marketplace-mint-nft.png)

## Displaying NFTs in the Home component
As we have already stored the data in our Redux store, we can display it in our Home component. We can first create a section component to render the NFTs.

Create a `Token_card.js` file in the `sections` directory. This component will be really simple without any logic. We will show details of our NFT here. Let us write this out.

We'll show:

- The image
- The amount
- The NFT's name
- The NFT's description
- A button to buy/collect the NFT

Here we can create prop `item` for the details, `onClick` to navigate to the detailed view and _onCollect_ which simply dispatches collectNFT action creator.

We will set up _onClick_ on the image and _onCollect_ to the buy button.

```js
const Token = ({ item, onClick, onCollect }) => {
	return (
		<div className="ui fluid card">
			<div className="image">
				<img
					onClick={onClick}
					style={{ maxHeight: "200px", objectFit: "cover" }}
					src={`https://ipfs.io/ipfs/${
						item.image.split("ipfs://")[1]
					}`}
					alt={item.description}
				/>
			</div>
			<div className="content">
				<div className="right floated">
					Price:
					<div style={{ color: "black" }}>{item.amount}</div>
				</div>
				<div className="header">{item.name}</div>
				<div className="meta">{item.symbol}</div>
				<div className="description">
					{item.description.length > 15
						? item.description.slice(0, 15) + "..."
						: item.description}
				</div>
			</div>

			<div className="extra content">
				<span className="right floated">
					<button className="ui basic button" onClick={onCollect}>
						{item.collectable ? "Buy" : "Sold Out"}
					</button>
				</span>
				<span>
					Token ID:
					<div style={{ color: "black" }}>{item.token_id}</div>
				</span>
			</div>
		</div>
	);
};

export default Token;
```

Now, we are all set to show the Tokens in our `Home.js` component.
First, we will import the `useDispatch` and `useSelector` hooks. We will use `useDispatch` to execute the collectNFT action creator and `useSelector` to show all the data from the store i.e. `state.tokenData`.

```js
import { useDispatch, useSelector } from "react-redux";
// ...
const Home = ({Tezos}) => {
	const selector = useSelector(state => state.tokenData);
    const dispatch = useDispatch();
```

We can also import `useHistory` from `react-router` to show the route on click.

```js
import { useHistory } from "react-router";
// ...
const Home = ({Tezos}) => {
    // ...
    const history = useHistory();
```

Now, we can run .map on the selector and return the `Token_card.js` component we just created. We pass in all the props that are required.

We can see the completed code here.

```js
import React from "react";
import { useDispatch, useSelector } from "react-redux";
import { useHistory } from "react-router";

import { collectNFT } from "../../actions";
import Token from "../sections/Token_card";

const Home = ({ Tezos }) => {
	const selector = useSelector((state) => state.tokenData);
	const dispatch = useDispatch();
	const history = useHistory();

	const tokens = selector.map((obj, idx) => (
		<Token
			key={idx}
			item={obj}
			onCollect={() =>
				dispatch(
					collectNFT({ Tezos, amount: obj.amount, id: obj.token_id })
				)
			}
			onClick={() =>
				obj.collectable && history.push(`/show/${obj.token_id}`)
			}
		/>
	));

	return <div className="ui link three column grid cards">{tokens}</div>;
};

export default Home;
```

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/Tezos-nft-marketplace-show-nft.png)

## Displaying the detailed view
To show the detailed view, we will use `useParams` hook from `react-router`. As we have already specified the id in the URL. We can retrieve it directly using this hook.

Using the id received we will filter the tokenData from our redux state. We can create a state `data` to set the filtered details.

```js
import React, { useEffect, useState } from "react";
import { useDispatch, useSelector } from "react-redux";
import { useParams } from "react-router";
import { collectNFT } from "../../actions";

const Show = ({ Tezos }) => {
	const selector = useSelector((state) => state.tokenData);
	const dispatch = useDispatch();
	const { id } = useParams();
	const [data, setData] = useState(null);

	useEffect(() => {
		const temp = selector[parseInt(id)];
		if (temp) {
			setData(temp);
		}
	}, [selector, id]);
```

Let us now show all the data. We will use a grid layout from sematic UI here. 

```js
return (
    <div className="ui internally celled grid">
        {data !== null ? (
            <>
                <div className="ui">{data.description}</div>
                <div className="row">
                    <div className="nine wide column">
                        <img
                            src={`https://ipfs.io/ipfs/${
                                data.image.split("ipfs://")[1]
                            }`}
                            alt={data.description}
                        />
                    </div>
                    <div className="seven wide column container center">
                        <div className="ui">
                            <h3 className="ui right floated header">
                                {data.name}
                            </h3>
                            <h3 className="ui left aligned header">Name</h3>
                        </div>
                        <div className="ui">
                            <h3 className="ui right floated header">
                                {data.symbol}
                            </h3>
                            <h3 className="ui left aligned header">
                                Symbol
                            </h3>
                        </div>
                        <div
                            className="ui "
                            onClick={() => {
                                navigator.clipboard.writeText(
                                    data.holder + ""
                                );
                            }}>
                            <h3
                                className="ui right floated header green"
                                style={{ cursor: "pointer" }}
                                data-content="Copy to clipboard">
                                {data.holder?.slice(0, 6) + "..."}
                            </h3>
                            <h3 className="ui left aligned header">
                                Holder
                            </h3>
                        </div>
                        <div
                            className="ui "
                            onClick={() => {
                                navigator.clipboard.writeText(
                                    data.author + ""
                                );
                            }}>
                            <h3
                                className="ui right floated header green"
                                style={{ cursor: "pointer" }}
                                data-content="Copy to clipboard">
                                {data.author?.slice(0, 6) + "..."}
                            </h3>
                            <h3 className="ui left aligned header">
                                Author
                            </h3>
                        </div>
                        <div className="ui">
                            <h3 className="ui right floated header">
                                {data.amount}
                            </h3>
                            <h3 className="ui left aligned header">
                                Price
                            </h3>
                        </div>
                        <div className="ui">
                            <h3 className="ui right floated header">
                                {data.token_id}
                            </h3>
                            <h3 className="ui left aligned header">
                                Token ID
                            </h3>
                        </div>
                        {/*...*/}
                    </div>
                </div>
            </>
        ) : null}
    </div>
	);
};

export default Show;

```

We have also imported collectNFT, We can give an option in the detailed view as well to Buy/collect the NFT. We will use `useDispatch` hook to dispatch the action creator.

```js
    <div className="ui">
        <button
            className="fluid ui button basic green"
            onClick={() =>
                data.collectable && 
                dispatch(
                    collectNFT({
                        Tezos,
                        amount: data.amount,
                        id: data.token_id,
                    })
                )
            }
        >
            {data.collectable?"Buy":"Sold Out"}
        </button>
    </div>
```

![image](https://github.com/figment-networks/datahub-learn/raw/master/assets/Tezos-nft-marketplace-show-detail-nft.png)

Hurray! We are all done. We have completed the UI of our NFT marketplace.
Check out the complete code on [Github](https://github.com/NishantChandla/nft-marketplace-tutorial)
Demo live on [Vercel](https://nft-marketplace-tutorial.vercel.app/)

# Conclusion
In this tutorial, you learned how to build an NFT marketplace on Tezos. We covered smart contract development with Smartpy. We built the User Interface with React.

There's still a lot of grounds to cover on this project. For instance, we are not handling errors. Also, we can create custom indexers with a backend to make fetching faster.

# About the Author
This tutorial was created by Nishant Chandla. You can get in touch with the author on [GitHub](https://github.com/NishantChandla)