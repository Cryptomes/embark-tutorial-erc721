## Listing the newly minted tokens
As you can see when you browser the DApp at this point, we show a placeholder spaceship, just so we can see how a token would look like when all the functionality of the DApp is implemented.

![Placeholder](https://raw.githubusercontent.com/status-im/status-dapp-workshop-mexico/tutorial-series/tutorial/images/shipPlaceholder.png)

Let's start by removing it. Open the file `app/js/index.js` and remove the constant `myShip` from `_loadMyShips()`. You also need to remove it from the `list` array.

Since we're going to use the `SpaceshipToken` contract, `EmbarkJS` and `web3` functions, let's import them

```
import EmbarkJS from 'Embark/EmbarkJS';
import web3 from "Embark/web3";
import SpaceshipToken from 'Embark/contracts/SpaceshipToken';
```

`EmbarkJS` provides a `onReady(err)` method to run Javascript code as soon as the connections to the web3 providers and ipfs are done and become safe to interact with. This function is ideal to perfom task that need to be executed as soon as these connections are made.

We'll use this function inside `componentDidMount()`, to load all the spaceships of the diferent sections of the DApp:

```
componentDidMount(){
    EmbarkJS.onReady((err) => {
        this._loadEverything();
    });
}
```

`_loadEverything()` has a call to the method `_loadShipsForSale()` which is the one we're interested to implement, in order to load all the newly minted tokens we wish to sell.

The `SpaceshipToken` contract has a couple of methods we can use for helping us display the tokens for sale and their information:

- `shipsForSaleN` returns the number of ships for sale in the store
- `shipsForSale` receives an index number and returns the token id for sale
- `spaceshipPrices` receives a token id, and return the price for that token
- `spaceships` which receives the id, and returns the struct that represents a spaceship.

Let's use these functions. First, let's extract them to their own variables, and declare an array to store the objects that will represent the spaceships:

```
_loadShipsForSale = async () => {
    const { shipsForSaleN, shipsForSale, spaceshipPrices, spaceships } = SpaceshipToken.methods;

    const list = [];
}
```

Since we cannot return all the spaceships for sale at once, we need a loop to query the information for each spaceship individually:

```
_loadShipsForSale = async () => {
    const { shipsForSaleN, shipsForSale, spaceshipPrices, spaceships } = SpaceshipToken.methods;

    const list = [];

    const total = await shipsForSaleN().call();
    if(total){
        for (let i = 0; i < total; i++) {
            const id = await shipsForSale(i).call();
            const info = await spaceships(id).call();
            const price = await spaceshipPrices(id).call();

            console.log(id);
            console.log(info);
            console.log(price);
        }
    }
}
```

The spaceship object for the store section requires the following attributes: `id`, `price`, `metadataHash`, `lasers`, `shield`, `energy`, and since the last four are part of the `info` variable, we can use the spread operator (...) to avoid having to write each attribute individually. Once you create the object, add it to `list`, and update the `shipsForSale` state with this array:

```
_loadShipsForSale = async () => {
    const { shipsForSaleN, shipsForSale, spaceshipPrices, spaceships } = SpaceshipToken.methods;

    const list = [];

    const total = await shipsForSaleN().call();
    if(total){
        for (let i = 0; i < total; i++) {
            const id = await shipsForSale(i).call();
            const info = await spaceships(id).call();
            const price = await spaceshipPrices(id).call();

            const ship = {
                id,
                price,       
                ...info
            };
            list.push(ship);
        }
    }
    this.setState({shipsForSale: list.reverse()});
}
```

![Spaceship](https://raw.githubusercontent.com/status-im/status-dapp-workshop-mexico/tutorial-series/tutorial/images/ship.png)


You may have noticed that the ships that you create are displayed in the UI, however, the image used is not the one we uploaded, and the prices are not formatted correctly. We need to work on the file `app/js/components/ship.js` for this.

As usual, start by importing `EmbarkJS` and `web3`, as well as our `SpaceshipToken`

```
import EmbarkJS from 'Embark/EmbarkJS';
import web3 from "Embark/web3";
import SpaceshipToken from 'Embark/contracts/SpaceshipToken';
```

The best place to load the image is when the component mounts. We will use `EmbarkJS.onReady` to invoke the `_loadAttributes` method

```
componentDidMount(){
    EmbarkJS.onReady((err) => {
        this._loadAttributes();
    });
}
```

The image URL is part of the token attributes stored in IPFS. We need to use the method `EmbarkJS.Storage.get` to obtain the content of the JSON object. We can access the IPFS hash via `this.props.metadataHash`. Remember we store this information in the contract as a `bytes` data type so we need to convert it back to string:

```
_loadAttributes(){
    const metadataHash = web3.utils.toAscii(this.props.metadataHash);
    EmbarkJS.Storage.get(metadataHash)
    .then(content => {
            console.log(content);
    });
}
```

The content returned is a string, so, before we can access the image attribute, we need to convert this string back to a JSON object using `JSON.parse(string)`. Once you do this, you are able to set the image state, and after reloading the DApp in the browser, the image should load.

```
_loadAttributes(){
    const metadataHash = web3.utils.toAscii(this.props.metadataHash);
    EmbarkJS.Storage.get(metadataHash)
    .then(content => {
        const jsonMetadata = JSON.parse(content);
        this.setState({image: jsonMetadata.image});
    });
}
```

Finally, to format the price, on the `render` method, we will update the constant `formattedPrice` to use `web3.utils.fromWei` and convert the value from wei to ether as expected:

```
const formattedPrice = !wallet ? web3.utils.fromWei(price, "ether") : '';
```


## Buying the tokens

![Buy](https://raw.githubusercontent.com/status-im/status-dapp-workshop-mexico/tutorial-series/tutorial/images/loadedShipImage.png)

As you can see, each spaceship has a buy button. Clicking this button should generate a transaction in which we send the value in wei of the token, and this token is transferred to our address. The `SpaceshipToken` contract has a `buySpaceship` function we can use for such purposes. 

Let's start by implementing the functionality of the `buyFromStore` method. Disable the button and extract the `buySpaceship` function to its own variable:

```
buyFromStore = () => {
    this.setState({isSubmitting: true});

    const { buySpaceship } = SpaceshipToken.methods;
}
```

The next step is to estimate the gas to invoke the contract function. For this scenario in particular, we need the token id (`this.props.id`), and, since this transaction involves sending ether, we need to specify the value. We can obtain it from `this.props.price`. Notice we also have a `catch` and `finally` blocks:

```
buyFromStore = () => {
    this.setState({isSubmitting: true});

    const { buySpaceship } = SpaceshipToken.methods;

    const toSend = buySpaceship(this.props.id)

    toSend.estimateGas({value: this.props.price })
    .then(estimatedGas => {
        console.log(estimatedGas);
    })
    .catch((err) => {
        console.error(err);
    })
    .finally(() => {
        this.setState({isSubmitting: false});
    });
}
```

Then, with the `estimatedGas` variable, we generate the transaction. Since buying the spaceship means that the ship will be transferred from the store to your wallet, it's a good idea to reload the different sections of the index page. You can use `this.props.onAction();` for this.

```
buyFromStore = () => {
    this.setState({isSubmitting: true});

    const { buySpaceship } = SpaceshipToken.methods;

    const toSend = buySpaceship(this.props.id)

    toSend.estimateGas({value: this.props.price })
    .then(estimatedGas => {
        return toSend.send({value: this.props.price,
                            gas: estimatedGas + 1000000});
    })
    .then(receipt => {
        console.log(receipt);
        this.props.onAction();
    })
    .catch((err) => {
        console.error(err);
    })
    .finally(() => {
        this.setState({isSubmitting: false});
    });
}
```

## Listing our tokens
This functionallity is very similar to loading the ships for sale. In fact you may even refactor it to avoid duplicating code. The functions that you need from the `SpaceshipToken` contract for this section are:

- `balanceOf`, which returns the number of tokens an address has. We will use `web3.eth.defaultAccount` since this contains the address that a user has when browsing our dapp.
- `tokenOfOwnerByIndex`, which expects an index number and will return the token id,
- and `spaceships` to retrieve the token information.

Open the file `app/js/index.js` and implement the method `_loadMyShips`: 

```
_loadMyShips = async () => {
    const { balanceOf, tokenOfOwnerByIndex, spaceships } = SpaceshipToken.methods;
    
    const list = [];

    const total = await balanceOf(web3.eth.defaultAccount).call();
    if(total){
        for (let i = total-1; i >= 0; i--) {
            const id = await tokenOfOwnerByIndex(web3.eth.defaultAccount, i).call();
            const info = await spaceships(id).call();
    
            const ship = {
              id,
              ...info
            };
            list.push(ship);
        }
    }
    this.setState({myShips: list.reverse()});
}
```

## Showing and withdrawing the balance from sales
When someone buys a spaceship from the store, the balance of the contract is incremented, and the owner should be able to query how much ether does the contract have, and also be able to withdraw this ether.

We will add this behavior to the WithdrawBalance component. Open the file `app/js/components/withdrawBalance.js`. We need to use web3 and EmbarkJS to add this functionality, so import those libraries:

```
import EmbarkJS from 'Embark/EmbarkJS';
import web3 from "Embark/web3";
import SpaceshipToken from 'Embark/contracts/SpaceshipToken';
```

The balance needs to be loaded when the page loads. This should be done in `componentDidMount()`. Remember to use `EmbarkJS.onReady((err) => {})` to be able to interact with the EVM as soon as the component mounts.

```
componentDidMount(){
    EmbarkJS.onReady((err) => {
        this._getBalance();
    });
}
```

`_getBalance()` will contain the logic required to query the balance. For this, we use `web3.eth.getBalance` to determine how much ether our `SpaceshipToken` contract address holds. This function returns a promise which resolves to a wei amount that needs to be formatted to Ether, since this is what the UI is expecting:

```
_getBalance(){
    web3.eth.getBalance(SpaceshipToken.options.address)
    .then(newBalance => {
            this.setState({
                balance: web3.utils.fromWei(newBalance, "ether")
            });
    });
}
```

![Balance](https://raw.githubusercontent.com/status-im/status-dapp-workshop-mexico/tutorial-series/tutorial/images/balance.png)

The withdraw button calls the `handleClick(e)` method, but at the moment it does not do anything. For implementing it's functionality we will use the function `withdrawBalance` of the `SpaceshipToken` contract.
The process is similar to what we have seen before: estimate gas before sending the transaction.

First, let's disable the button when we click it, and extract the `withdrawBalance` function to its own variable:

```
handleClick(e){
    e.preventDefault();
    this.setState({isSubmitting: true});

    const { withdrawBalance } = SpaceshipToken.methods;
}
```

Then, proceed to estimate the cost of calling `withdrawBalance()` using `estimateGas()`. Remember this returns a promise which resolves to the gas cost. It's also a good idea to add a `catch` and `finally` blocks to handle errors and enable the withdraw button again.

```
handleClick(e){
    e.preventDefault();
    this.setState({isSubmitting: true});

    const { withdrawBalance } = SpaceshipToken.methods;

    const toSend = withdrawBalance();
    toSend.estimateGas()
    .then(estimatedGas => {
        console.log(estimatedGas);
    })
    .catch((err) => {
        console.error(err);
    })
    .finally(() => {
        this.setState({isSubmitting: false});
    });
}
```

Finally, once we have the estimated gas, we can submit the transaction. As we did before, we will add a little extra gas to deal with unprecise gas estimations. Also, after we get the receipt, we will query again the balance in case it has changed between the withdrawal transaction and the current block.

```
handleClick(e){
    e.preventDefault();
    this.setState({isSubmitting: true});

    const { withdrawBalance } = SpaceshipToken.methods;

    const toSend = withdrawBalance();
    toSend.estimateGas()
    .then(estimatedGas => {
        return toSend.send({gas: estimatedGas + 1000});
    })
    .then(receipt => {
        console.log(receipt);
        this._getBalance();
    })
    .catch((err) => {
        console.error(err);
    })
    .finally(() => {
        this.setState({isSubmitting: false});
    });
}
```

## Display the mint form only if we are the owners of the contract
One thing you may have noticed is that the minting form shows up always even if you're not the owner of the contract. Our `SpaceshipToken` contract inherits from `Owned` which adds an `owner()` method that we can use to determine if the account browsing the dapp is the owner of the contract

For this, let's go back to `app/js/index.js`. The method `_isOwner` needs to be called when the component mounts, due to it being a good place to load data from a remote endpoint (in this case, the EVM). Also, it's a good idea to set the default state to `false` in the constructor

```
constructor(props) {
    super(props);
    this.state = {
        isOwner: false,
        hidePanel: false,
        shipsForSale: [],
        myShips: [],
        marketPlaceShips: []
    }
}
```

```
componentDidMount(){
    EmbarkJS.onReady((err) => {
        this._isOwner();
        this._loadEverything();
    });
}
```

```
_isOwner(){
    SpaceshipToken.methods.owner()
        .call()
        .then(owner => {
            this.setState({isOwner: owner == web3.eth.defaultAccount});
            return true;
        });
}
```

> `web3.eth.defaultAccount` is always used by default in the `from` property when sending/calling a contract function. If you use the Status app, or Metamask, it will contain the account you use to create transactions.

