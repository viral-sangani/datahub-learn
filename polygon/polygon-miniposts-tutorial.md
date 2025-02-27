# Introduction

In this tutorial, you will learn how to build a simple blogging website on Polygon (Matic). The dapp allows users to visualize their name, biography and the feed of all the posts. The user can also publish posts.

We will learn some basic elements of Solidity such as mapping or struct and how to deploy it on the Polygon Mumbai test network. For the frontend, we will use Reactjs.

# Prerequisites

- [Polygon pathway](https://learn.figment.io/protocols/polygon)
- Familiarity with Metamask, Solidity and Javascript are recommended.

# Requirements

- [Tutorial’s Github repo](https://github.com/aeither/miniblog)
- [Remix Ethereum](https://remix.ethereum.org/) Browser Solidity editor
- [Metamask](https://metamask.io/) Browser extension
- [Code Editor](https://code.visualstudio.com/download) For example VS Code
- [GitHub account](https://github.com/)

It is strongly recommended to create a new MetaMask account for testing. You will want to keep the Secret Recovery Phrase for this fresh account handy because it is needed for deployment of the smart contract.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts05.png)

Now you'll notice zero balance (0 MATIC) in your wallet, to get test Matic for deployment and testing go to the Matic Faucet [link](https://faucet.matic.network) -> Select Mumbai -> Paste your wallet address -> Click "Submit".

When this is complete, check your Metamask & you'll see some MATIC tokens there. We only need a small amount of MATIC to deploy and test our dApp.

# Create the contract

The first thing that we are going to accomplish is to create a working smart contract that allows everyone to post a twitter-like post. Whoever could interact with the smart contract to get their message published where each message has its id.

Open your browser and go to [Remix](https://remix.ethereum.org/)
This is one of the best online editors for development with Solidity. You will see several folders that you can delete as we won't use them.
Press the **Create New File** icon to create a new Solidity file named `miniblog.sol` at the root.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts01.png)

At the time of writing this tutorial it is advisable to use a Solidity version above 0.8.0. A cutting edge version could contain unknown issues and older versions would force us to deal with old and deprecated code.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
```

We need two structs to define the structure of the data we want to store. The first one will contain the user information and the second one the post information.

```solidity
contract MiniBlog {

   struct User {
       string username;
       string bio;
       uint256[] postIds;
   }

   struct Post {
       string title;
       string content;
       address author;
       uint256 created;
   }

}
```

Mapping in Solidity acts like a hashtable or dictionary in any other language. These are used to store the data as key-value pairs. We create 2 `mapping`, an `address` to User data and a postId to post Info.

```solidity
  mapping (address => User) public users;
  mapping (uint256 => Post) public posts;
```

With the mapping, we can access post data such as title or content with the id as key. For example `posts[2].title`.

Add the post id that we can increment and assign to each new post.

```solidity
contract MiniBlog {
  // Add latestPostId Here
  uint256 public latestPostId = 0;

  struct User {
    string username;
    string bio;
    uint256[] postIds;
    }
  // ...
}
```

Below mapping we create a function that takes title and content as parameter

```solidity
   function createPost(string memory title, string memory content) public {
       latestPostId++;

       posts[latestPostId] = Post(title, content, msg.sender, block.timestamp);
       users[msg.sender].postIds.push(latestPostId);
   }
```

When the function is called, it creates a new post and pushes a unique incremental Id into the array used in the mapping for posts. Then we push the id of the post to the users mapping for the caller of the function (msg.sender) so later we can track all the posts created by that writer.

With require, we can restrict to only the writer of the post who can edit his posts.

```solidity
 function modifyPostTitle(uint256 postId, string memory title) public {
   require(msg.sender == posts[postId].author, "Only the author can modify");

   posts[postId].title = title;
 }

 function modifyPostContent(uint256 postId, string memory content) public {
   require(msg.sender == posts[postId].author, "Only the author can modify");

   posts[postId].content = content;
 }
```

The caller can customize his profile by changing the username and the biography.

```solidity
 function updateUsername(string memory username) public {
   users[msg.sender].username = username;
 }

 function updateBio(string memory bio) public {
   users[msg.sender].bio = bio;
 }
```

To get the posts data we have created 2 functions. One gets an array of all the post Ids created by the same address. Another to get the post Id content. For the `getPostById`, we can access directly through the mapping posts variable as we declared it as `public`.

```solidity
function getPostIdsByAuthor(address author)
   public
   view
   returns (uint256[] memory)
 {
   return users[author].postIds;
 }

 function getPostById(uint256 postId)
   public
   view
   returns (string memory title, string memory content)
 {
   title = posts[postId].title;
   content = posts[postId].content;
 }
```

In Solidity, events are dispatched signals the smart contracts can fire. With the function `createPost`, we can add the `emit` keyword to trigger an event.

```solidity
 function createPost(string memory title, string memory content) public {
   latestPostId++;

   posts[latestPostId] = Post(title, content, msg.sender, block.timestamp);
   users[msg.sender].postIds.push(latestPostId);

   emit NewPost(msg.sender, latestPostId, title);
 }
```

We declare the event to be triggered. you can see the events of the smart contracts on Etherscan, in the events section. Example: https://etherscan.io/address/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#events

```solidity
event NewPost(address indexed author, uint256 postId, string title);
```

Now putting it all together, we should have the following contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MiniBlog {
 uint256 public latestPostId = 0;

 struct User {
   string username;
   string bio;
   uint256[] postIds;
 }

 struct Post {
   string title;
   string content;
   address author;
   uint256 created;
 }

 mapping(address => User) public users;
 mapping(uint256 => Post) public posts;

 event NewPost(address indexed author, uint256 postId, string title);

 function createPost(string memory title, string memory content) public {
   latestPostId++;

   posts[latestPostId] = Post(title, content, msg.sender, block.timestamp);
   users[msg.sender].postIds.push(latestPostId);

   emit NewPost(msg.sender, latestPostId, title);
 }

 function modifyPostTitle(uint256 postId, string memory title) public {
   require(msg.sender == posts[postId].author, "Only the author can modify");

   posts[postId].title = title;
 }

 function modifyPostContent(uint256 postId, string memory content) public {
   require(msg.sender == posts[postId].author, "Only the author can modify");

   posts[postId].content = content;
 }

 function updateUsername(string memory username) public {
   users[msg.sender].username = username;
 }

 function updateBio(string memory bio) public {
   users[msg.sender].bio = bio;
 }

 function getPostIdsByAuthor(address author)
   public
   view
   returns (uint256[] memory)
 {
   return users[author].postIds;
 }

 function getPostById(uint256 postId)
   public
   view
   returns (string memory title, string memory content)
 {
   title = posts[postId].title;
   content = posts[postId].content;
 }
}
```

On the Remix site, go to Solidity Compiler and click the blue button with compiler 0.8.0:

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts02.png)

This should successfully compile the smart contract. There would be a green icon in the left tab. Go to **Deploy & Run Transactions**. Make sure the Environment is set to **JavaScript VM**. Deploy the contract.
You should now see the contract under deployed contracts.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts03.png)

You can play with the contract. Update your username, create several posts, change them and check the result by getting the posts.

Outstanding! We have created a mini-blogging smart contract. Now let’s build the frontend for this.

# Create the frontend

We are going to use a created repository to make it easier to follow.
In this repository there are 3 branches, you can checkout from 1 to 3 to see the progress and changes while following this tutorial.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts04.png)

Clone the repository by running git clone in your terminal.

```text
git clone https://github.com/aeither/miniblog
cd miniblog
```

Checkout to the first branch with the command:

```text
git checkout 1-web3-signin
```

Open the terminal and run yarn then yarn dev to run the project locally. We use react-moralis to allow the user to log in with Metamask.

Import useMoralis.

```tsx
// 1 Import useMoralis
import { useMoralis } from "react-moralis";

// ...

// 2 Use constants from Moralis
const { authenticate, isAuthenticated, user, logout } = useMoralis();
```

If the user is not authenticated we show the authenticate button. If logged we show the user his address and the log out button.

```tsx
// 3 Add Login Bar
const LoginBar = () =>
  !isAuthenticated ? (
    <Button onClick={() => authenticate()}>Authenticate</Button>
  ) : (
    <HStack
      rounded="8"
      background={GRAY}
      w="100%"
      p="4"
      my="4"
      justify="space-between"
    >
      <Box>
        <Heading size="sm">User ID: {user?.get("username")}</Heading>
        <Text>Address: {user?.get("ethAddress")}</Text>
      </Box>
      <Button onClick={() => logout()}>Logout</Button>
    </HStack>
  );
```

With Mumbai network selected in Metamask. In Remix deploy this time with environment set to Injected Web3

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts06.png)

When you hit deploy, Metamask should show a confirmation for the transaction.

Now we need the ABI and the smart contract address.
In the compiler, you can go to the end and copy the ABI. an application binary interface (ABI) is an interface between two binary program modules. It allows us to interact with the contract.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts07.png)

Back to your editor, inside the src folder, we create an abi.json file and paste the ABI.

Get the contract address from deployed contracts in Remix or https://mumbai.polygonscan.com/

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts08.png)

Import the ABI and add the smart contract address.

```tsx
// 1 Import ABI and contract address
import contractABI from "../abi.json";

const address = "ADDRESS";
```

Import Moralis from useMoralis()

```tsx
async function initContract() {
  const web3 = await Moralis.Web3.enable();
  const contract = new web3.eth.Contract(contractABI, address);
  setContract(contract);
}
// 3.2 Get user profile
async function getUserProfile() {
  if (contract !== undefined) {
    const profileData = await contract.methods.users(userAddress).call();
    setUserProfile({
      bio: profileData.bio,
      username: profileData.username,
    });
  }
}

// 3.3 Get latest posts
async function getPosts() {
  if (contract !== undefined) {
    const latestPostId = await contract.methods.latestPostId().call();
    // We only want the last 10 posts
    const last = latestPostId < 10 ? 0 : 10;
    const posts: Array<Post> = [];
    // eslint-disable-next-line no-plusplus
    for (let index = latestPostId; index > last; index--) {
      // eslint-disable-next-line no-await-in-loop
      const post = await contract.methods.posts(index).call();
      posts.push({
        author: post.author,
        content: post.content,
        created: post.created,
        title: post.title,
      });
    }
    setLatestPosts(posts);
  }
}
```

Inside `initContract` we enabled the web3 provider and created a new contract. `getUserProfile` makes a call to function users which returns the user bio and name. We can go to remix and add the name and bio there as we won’t cover the interaction from the front end. `getPosts` return the latest 10 posts and set to state.

Check out the third branch to learn more about how to interact with the contract.

We added **react-hook-form** as a dependency already so we can use it inside `NewPostForm`. Run `yarn` to install it or `yarn add react-hook-form` if it is not already installed in your project.

```tsx
<form onSubmit={handleSubmit(onSubmit)}>
  <FormControl>
    <FormLabel htmlFor="title">Title</FormLabel>
    <Input id="title" placeholder="title" {...register("title")} />
    <FormLabel htmlFor="content">Content</FormLabel>
    <Input id="content" placeholder="content" {...register("content")} />
  </FormControl>
  <Button mt={4} colorScheme="teal" isLoading={isSubmitting} type="submit">
    Submit
  </Button>
</form>
```

There are 2 available fields in the form, the title and the content. When the user submits the form, it will run the following function.

```tsx
const onSubmit: SubmitHandler<IFormInput> = ({
  title,
  content,
}: IFormInput) => {
  contract?.methods
    .createPost(title, content)
    .send({ from: userAddress })
    .then((receipt: unknown) => {
      console.log(receipt);
    });
};
```

This time as you can observe, we use send instead of call. This transaction should be signed by an address so the parameter from: address is mandatory.

Now let’s see the result. \*Make sure you are in Mumbai. Make sure you have the dependencies install with `yarn` or `npm install` and run with `yarn dev`.

We should be able to see the homepage by visiting http://localhost:3000. Authenticate with your Metamask extension.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts09.png)

We can create the first posts by going to the plus icon and submit a new post.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts10.png)

If we update the username or modify the post from remix you can see the changes by refreshing the frontend.

![](https://github.com/figment-networks/datahub-learn/raw/master/assets/miniposts11.png)

Awesome. We made it. Now you have a mini post web app from where you can bootstrap your journey in the Matic network.

# Conclusion

In this tutorial we made a mini posts Web 3 application. We learned some of the basics of Solidity: How to create a smart contract, how to deploy it, how to interact with it, mapping, struct and events. Then we built the frontend where we see how to interact with the contract with a web app. We have played a little with web3js from Moralis.

# About the Author

Giovanni Fu is a web developer, blockchain enthusiast with a passion for Software Development, & Decentralization. Feel free to connect with me on [GitHub](https://github.com/aeither).

# References

- Web3 docs: https://web3js.readthedocs.io/en/v1.5.2/
- Polygon (Matic) docs: https://docs.matic.network/docs/develop/getting-started
- MetaMask docs: https://docs.metamask.io/guide/#why-metamask
- React docs: https://reactjs.org/docs/getting-started.html
- Moralis docs: https://docs.moralis.io/moralis-server/tools/react-moralis
- Frontend Repo: https://github.com/aeither/miniblog
- Starter Template: https://github.com/sozonome/nextarter-chakra
