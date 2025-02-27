# Introduction

This is the second part of two-part tutorial series on how to create a Peer-to-Peer payment dApp. In the first part, we created an ERC20 token using an openzappline contract and deployed it on the Polygon testnet. This part will learn how to add other tokens like USDT, BUSD, etc... on the Polygon network.

In the last part, we deployed our smart contract on the Polygon testnet. To add more tokens to our dApp, we need other reliable tokens available in Polygon Mainnet. So we will start by deploying our Payment token to the Polygon mainnet and then use other tokens already available on the Polygon mainnet in our dApp to make payments. By doing so, we will be shifting our dApp from Polygon testnet to Polygon mainnet.

Before starting, let's see what our dApp will look like after adding support for multiple tokens.

![App demo](https://github.com/figment-networks/datahub-learn/raw/master/assets/payment-gateway-part-2.gif)

# Prerequisites

To successfully follow along with this tutorial, you will need an understanding of ERC20 tokens, basic knowledge of the frontend framework (NextJS in our case), and you will need to complete the first part of this tutorial, at <https://learn.figment.io/tutorials/peer-to-peer-payment-on-polygon-part-1>.

# Requirements

- Metamask - You will need a Metamask wallet with a Polygon mainnet configured to deploy our smart contract and use our dApp.
- Figment Datahub API - We will be using DataHub's Polygon Mainnet RPC URL to deploy the smart contract.

# Deploying smart contract to Polygon Mainnet

To deploy our contract, we need Datahub's Polygon Mainnet RPC URL. You can find all the Polygon URLs from Datahub's dashboard.

Open the `truffle-config.js` and add the `mainnet` configuration to the network section of the configuration.

```javascript
...
networks: {
    development: {
      host: "localhost",
      port: 7545,
      network_id: "*",
    },
    matic: {
      provider: () =>
        new HDWalletProvider(
          mnemonic,
          `https://matic-mumbai--rpc.datahub.figment.io/apikey/${process.env.DATAHUB_POLYGON_API}/`
        ),
      network_id: 80001,
      confirmations: 2,
      timeoutBlocks: 200,
      skipDryRun: true,
      chainId: 80001,
    },
    mainnet: {
      provider: () =>
        new HDWalletProvider(
          mnemonic,
          `https://matic-mainnet--jsonrpc.datahub.figment.io/apikey/${process.env.DATAHUB_POLYGON_API}/`
        ),
      network_id: 137,
      confirmations: 2,
      timeoutBlocks: 200,
      skipDryRun: true,
      chainId: 137,
    },
...
```

*Note*: Since we are deploying to mainnet, we need at least 0.01 MATIC token present in our account to pay for deployment gas fees. These MATIC tokens are not the same tokens we use on the testnet, and we cannot get these tokens from the Polygon faucet. The faucet is only for testnet tokens. To get MATIC tokens on the Polygon mainnet, you need to buy the token from an exchange and transfer those tokens to your Metamask wallet.

To deploy the contract to the mainnet, run the following command.

```text
truffle deploy --network mainnet --reset
```

Truffle will deploy the contract on Polygon mainnet, and since we have sent the `10000 PAY` token to the address that deployed the contract, we should have received `10000 PAY` tokens in our wallet.

# Adding Payment token to Metamask

Adding a custom token to Metamask is pretty simple; all you need is the token contract address. The contract address of the Payment token can be fetched from the output of the `truffle deploy` command or from the network section of the `abis/PaymentToken.json` file.

Open Metamask, and make sure you are connected to the Polygon mainnet network. In the assets section, click on the Import tokens button, and paste the contract address of the token you are trying to add, in our case the address of the PAY token. Metamask will automatically fetch the token symbol and token decimals from the contract if the token address is correct.

![Metamask Import Token](https://github.com/figment-networks/datahub-learn/raw/master/assets/metamask-import-token.jpg)

![Metamask Import Token form](https://github.com/figment-networks/datahub-learn/raw/master/assets/metamask-import-token-form.png)

# Adding logic for multiple token

Before we start working on `DataContext` that we created in the past one of this tutorial, we need to add a couple of dependencies to our NextJS app. Fire up the terminal and run the following command:

```text
yarn add @maticnetwork/meta @headlessui/react
```

`@maticnetwork/meta` is the official package created by the maticnetwork team to get the static content related to the Polygon network. This includes the default RPC URLs and the contract address of all the tokens deployed by maticnetwork (Polygon, ERC20, TESTERC20, etc..) on both mainnet and testnet along with ABIs for the default ERC20 token. Since we are adding multiple ERC20 tokens in our dApp and usually, all the ERC20 tokens have the same ABI, we can use the ABI from the `@maticnetwork/meta` package when making standard calls like `balanceOf` and `transfer`.

`@headlessui/react` is an unstyled, fully accessible UI component designed to integrate with Tailwind CSS.

Let's start by modifying `DataContext.tsx`. Replace the imports and interface for `DataContext` with the code below.

```javascript
declare let window: any;
import Network from "@maticnetwork/meta/network";
import { createContext, useContext, useState } from "react";
import Web3 from "web3";

interface DataContextProps {
  account: string;
  loading: boolean;
  loadWallet: () => Promise<void>;
  sendPayment: ({
    amount,
    toAddress,
  }: {
    amount: any;
    toAddress: any;
  }) => Promise<any>;
  balance: number;
  selectedToken: Token;
  updateSelectedToken: (token: Token) => void;
}
```

You'll notice that we have removed the import for the `PaymentToken` ABI because we will be using the ABI for ERC20 tokens from `@maticnetwork/meta/network`. In the interface, we have added two new variables, `selectedToken` and `updateSelectedToken`.

In `useProviderData`, replace the state variables with the following code:

```javascript
  const [loading, setLoading] = useState(true);
  const [account, setAccount] = useState<string>();
  const [balance, setBalance] = useState<number>();
  const [selectedToken, setSelectedToken] = useState<Token>(tokensList[0]);
  const [erc20Abi, setErc20Abi] = useState<any>();
```

We have removed the `paymentToken` variable and added two new variables, `selectedToken` and `erc20Abi`. `selectedToken` will contain the current token that the user has selected to make payment, and `erc20Abi` will have the ABI json for standard ERC20 token.

Now, update the `loadWallet` function with the below code:

```javascript
  const loadWallet = async () => {
    if (window.ethereum) {
      const network = new Network("mainnet", "v1");
      const ERC20ABI = network.abi("ERC20");
      setErc20Abi(ERC20ABI);

      window.web3 = new Web3(window.ethereum);
      await window.ethereum.enable();
      const web3 = window.web3;
      window.ethereum.on("accountsChanged", function (accounts) {
        loadWallet();
      });
      var allAccounts = await web3.eth.getAccounts();
      setAccount(allAccounts[0]);

      var paymentTokenInstance = new web3.eth.Contract(
        ERC20ABI,
        "0xA82EAf2c0e8100ECaB913A601f652F7C4151c549"
      );
      var bal = await paymentTokenInstance.methods
        .balanceOf(allAccounts[0])
        .call();
      setBalance(bal);

      setLoading(false);
    } else {
      window.alert("Non-Eth browser detected. Please consider using MetaMask.");
    }
  };
```

We create an instance of `Network` and fetch ERC20 ABI from it, then set the state variable `erc20Abi`. Next, we bring the account from metamask and get the `Pay` token balance for that account. To get the balance, we create an instance of `paymentToken` using `ERC20ABI` and contract address from the Polygon mainnet and call the `balanceOf` method to get the current balance of the user's address.

Let's update the `sendPayment` function.

```javascript
  const sendPayment = async ({ amount, toAddress }) => {
    try {
      var amountInDecimal;
      if (selectedToken.decimals === 18) {
        amountInDecimal = window.web3.utils.toWei(amount, "ether");
      } else {
        amountInDecimal = amount * Math.pow(10, selectedToken.decimals);
      }
      var tokenContract = new window.web3.eth.Contract(
        erc20Abi,
        selectedToken.address
      );

      var bal = await tokenContract.methods.balanceOf(account).call();
      if (bal < amountInDecimal) {
        return "You don't have enough balance";
      }
      const txHash = await tokenContract.methods
        .transfer(toAddress, amountInDecimal)
        .send({
          from: account,
        });
      setTimeout(async () => {
        var bal = await tokenContract.methods.balanceOf(account).call();
        setBalance(bal);
      }, 2000);
      return "Payment success";
    } catch (e) {
      return e.message;
    }
  };
```

To make payment, we have to convert the amount entered by the user to their respective decimal places. We have not yet defined the tokens list and decimals for that token. If the token has 18 decimals, then we can directly use `web3.utils.toWei(amount, "ether")` to convert the amount, but in some cases, like stable coins that only have six decimals, we cannot use `toWei,` and we have to convert the amount manually. So in `sendPayment`, we are first checking the decimals and then converting the amount to decimals.

Next, we create an instance of the selected token contract and check the balance of the current account. If the balance is less than the amount requested to transfer, we return the error string. If the amount is valid, then we call the `transfer` method of the ERC20 contract and make a transaction from the current account to the address entered by the user.

After the transfer is completed, we create a timeout for 2 seconds and fetch the updated balance for the selected token. The 2 seconds delay is added to compensate for any delay that occurred in the transaction.

Before moving forward, let's create an interface for Tokens and created a tokens list that the user can choose from to make payment. In `DataContext.tsx`, add the following code after `useProviderData` function.

```javascript
export interface Token {
  name: string;
  symbol: string;
  address: string;
  logo: string;
  decimals: number;
}

export const tokensList: Token[] = [
  {
    name: "Payment Token",
    symbol: "PAY",
    address: "0xA82EAf2c0e8100ECaB913A601f652F7C4151c549",
    logo: "",
    decimals: 18,
  },
  {
    name: "Chainlink",
    symbol: "LINK",
    address: "0x53E0bca35eC356BD5ddDFebbD1Fc0fD03FaBad39",
    logo: "https://gemini.com/images/currencies/icons/default/link.svg",
    decimals: 18,
  },
  {
    name: "Wrapped Bitcoin",
    symbol: "wBTC",
    address: "0x1BFD67037B42Cf73acF2047067bd4F2C47D9BfD6",
    logo: "https://raw.githubusercontent.com/trustwallet/assets/master/blockchains/ethereum/assets/0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599/logo.png",
    decimals: 8,
  },
  {
    name: "Wrapped Ethereum",
    symbol: "wETH",
    address: "0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619",
    logo: "https://gemini.com/images/currencies/icons/default/link.svg",
    decimals: 18,
  },
  {
    name: "Tether USD",
    symbol: "USDT",
    address: "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
    logo: "https://raw.githubusercontent.com/Uniswap/assets/master/blockchains/ethereum/assets/0xdAC17F958D2ee523a2206206994597C13D831ec7/logo.png",
    decimals: 6,
  },
  {
    name: "Binance USD",
    symbol: "BUSD",
    address: "0xdAb529f40E671A1D4bF91361c21bf9f0C9712ab7",
    logo: "https://raw.githubusercontent.com/Uniswap/assets/master/blockchains/ethereum/assets/0x4Fabb145d64652a948d72533023f6E7A623C7C53/logo.png",
    decimals: 6,
  },
  {
    name: "Aave",
    symbol: "AAVE",
    address: "0xD6DF932A45C0f255f85145f286eA0b292B21C90B",
    logo: "https://gemini.com/images/currencies/icons/default/aave.svg",
    decimals: 18,
  },
];

```

We have created an interface called `Token`, which has the name, symbol, address, logo image URL, and decimals of that token. Next, we have created `tokenList`, which contains some standard tokens on Polygon mainnet network. You can get contracts addresses and basic information of all tokens from [Polygon Token Mapper](https://mapper.matic.today/) site.

The last thing to do in `DataContext` is to create a function to update the `selectedToken`. Add the following code in `useProviderData`:

```javascript
  const updateSelectedToken = async (token: Token) => {
    var tokenContract = new window.web3.eth.Contract(erc20Abi, token.address);
    var bal = await tokenContract.methods.balanceOf(account).call();
    setBalance(bal);
    setSelectedToken(token);
  };
```

When the user changes the token from UI, we need to fetch the balance of that token and set the balance to update the UI.

# Updating the UI

We will create a modal for the token selection UI that users can use to shift between the tokens quickly. To create UI, we will use `@headlessui/react`. Create a file `TokenModal.tsx` in the components folder and paste the following code.

```javascript
import { Dialog, Transition } from "@headlessui/react";
import React, { Fragment } from "react";
import { tokensList, useData } from "../contexts/DataContext";

interface TokensModalProps {
  isOpen: boolean;
  closeModal: () => void;
  openModal: () => void;
}

const TokensModal: React.FC<TokensModalProps> = ({
  children,
  closeModal,
  isOpen,
}) => {
  const { updateSelectedToken } = useData();
  return (
    <>
      {children}
      <Transition appear show={isOpen} as={Fragment}>
        <Dialog
          as="div"
          className="fixed inset-0 z-10 overflow-y-auto"
          onClose={closeModal}
        >
          <div className="min-h-screen px-4 text-center">
            <Transition.Child
              as={Fragment}
              enter="ease-out duration-300"
              enterFrom="opacity-0"
              enterTo="opacity-100"
              leave="ease-in duration-200"
              leaveFrom="opacity-100"
              leaveTo="opacity-0"
            >
              <Dialog.Overlay className="fixed inset-0" />
            </Transition.Child>

            {/* This element is to trick the browser into centering the modal contents. */}
            <span
              className="inline-block h-screen align-middle"
              aria-hidden="true"
            >
              &#8203;
            </span>
            <Transition.Child
              as={Fragment}
              enter="ease-out duration-300"
              enterFrom="opacity-0 scale-95"
              enterTo="opacity-100 scale-100"
              leave="ease-in duration-200"
              leaveFrom="opacity-100 scale-100"
              leaveTo="opacity-0 scale-95"
            >
              <div className="inline-block w-full max-w-md p-6 my-8 overflow-hidden text-left align-middle transition-all transform bg-gray-700 border border-gray-600 shadow-xl rounded-2xl">
                <Dialog.Title
                  as="h3"
                  className="text-lg font-medium leading-6 text-white"
                >
                  Select Token
                </Dialog.Title>
                <div className="flex flex-col mt-4 space-y-1">
                  {tokensList.map((item) => {
                    return (
                      <div
                        className="flex flex-row items-center space-x-3 hover:bg-gray-600 py-2 px-3 rounded-2xl cursor-pointer"
                        onClick={() => {
                          updateSelectedToken(item);
                          closeModal();
                        }}
                      >
                        {item.logo ? (
                          <img
                            className="w-7 h-7 border rounded-full"
                            src={item.logo}
                          />
                        ) : (
                          <div className="w-7 h-7 border rounded-full text-white flex justify-center items-center font-bold">
                            {item.symbol[0]}
                          </div>
                        )}
                        <div className="flex flex-col">
                          <span className="text-lg leading-5 font-medium text-white">
                            {item.symbol}
                          </span>
                          <span className="text-xs leading-5 text-white">
                            {item.name}
                          </span>
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>
            </Transition.Child>
          </div>
        </Dialog>
      </Transition>
    </>
  );
};

export default TokensModal;

```

We are iterating over the `tokenList` and creating a list of tokens for the user to select from. When the user clicks on any token, we will call the `updateSelectedToken` from the `useData` context and pass in the token that the user clicked on, and close the modal. You can refer to [this site](https://headlessui.dev/react/dialog) to learn more about headlessUI modals.

This is how the modal looks like:

![Tokens Modal](https://github.com/figment-networks/datahub-learn/raw/master/assets/tokens-modal.png)

Now that we are done with the modal, let's update our `index.tsx` to replace **Pay Token** with the modal button.

In `index.tsx` replace this:

```javascript
<div className="px-3 py-2 bg-gray-800 rounded-2xl flex flex-row items-center">
  <span className="text-white text-lg font-bold">
    PAY Token
  </span>
</div>
```

With this:

```javascript
<TokensModal
closeModal={closeModal}
isOpen={isOpen}
openModal={openModal}
>
  <div
    className="px-3 py-2 bg-gray-800 rounded-2xl flex flex-row items-center cursor-pointer"
    onClick={openModal}
  >
    {selectedToken.logo ? (
      <img
        className="w-6 h-6 border rounded-full"
        src={selectedToken.logo}
      />
    ) : (
      <div className="w-6 h-6 border rounded-full text-white flex justify-center items-center font-bold">
        {selectedToken.symbol[0]}
      </div>
    )}
    <span className="text-white text-lg font-bold mx-2">
      {selectedToken.symbol}
    </span>
    <svg
      xmlns="http://www.w3.org/2000/svg"
      className="h-5 w-5"
      viewBox="0 0 20 20"
      fill="white"
    >
      <path
        fillRule="evenodd"
        d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z"
        clipRule="evenodd"
      />
    </svg>
  </div>
</TokensModal>
```

Here instead of showing a button, we are creating a button to open the modal. On the button, we are displaying the selected token symbol and token logo. If the token logo is not available, we show the first token letter to make the placeholder.

Now we need to update the way we show the balance. Before, we were only showing PAY token balance, which has 18 decimals, we can easily use `Web3.utils.fromWei` to show the balance, but now that we have some tokens with the different decimal values, we need to update the code.

Replace this code:

```javascript
{balance &&
  `Balance: ${Web3.utils.fromWei(
    balance.toString(),
    "ether"
  )} PAY`}
```

With this:

```javascript
{balance &&
  `Balance: ${
    selectedToken.decimals === 18
      ? Web3.utils.fromWei(balance.toString(), "ether")
      : balance / Math.pow(10, selectedToken.decimals)
  } ${selectedToken.symbol}`}
```

Now the last thing remaining is the way to open and close the modal. Create the following state variables and functions in the `Home` component.

```javascript
  let [isOpen, setIsOpen] = useState(false);

  function closeModal() {
    setIsOpen(false);
  }

  function openModal() {
    setIsOpen(true);
  }
```

The source code is available [here](https://github.com/viral-sangani/polygon-peer-to-peer-payment).

# Conclusion

Congratulation on finishing part 2 of this tutorial. Thank you for taking the time to complete it. In this tutorial, we have learned how to add multi-token support to our payment dApp.

Keep building on Web3.

# About the Author

I'm Viral Sangani, a tech enthusiast working on blockchain projects & love the Web3 community. Feel free to connect with me on [Github](https://github.com/viral-sangani).
