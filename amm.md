#Introduction to XRPL

## getting started

Before anything else, you are going to want to setup your coding environment.

Any javascript editor will do and feel free to use your own node environement to follow along with javascript. Most of the code we will use can be run client side as well.

If you want a hastle free setup, use replit (https://replit.com/). Accounts are free and then create a new replit with the typescript defaults or the node defaults if you would rather avoid typescript.

If you use another language, you can follow along using a script of your own, but you will have to use the appropriate SDK which may differ from xrpl.js at times.

You will often want to refer to the main API documentation for transactions here: https://xrpl.org/docs/references/protocol/transactions/types/

### faucets

While faucets are accessible programatically, you can also create a test account via this page: https://xrpl.org/resources/dev-tools/xrp-faucets/

### wallets

This tutorial makes use of the Xaman wallet, which you can download from your mobile phone at https://xumm.app. Once installed you can import a wallet via "family seed" (the address secret) for quicker setup.

Most examples in this tutorial sign transactions programmatically however.

creating the main structure of your script

create a new file or edit index.ts

```
import {  Client,  Wallet }  from "xrpl" 

const client = new Client("wss://s.altnet.rippletest.net:51233")


const main = async () => {
  console.log("lets get started...");
  await client.connect();

  // do something interesting here


  await client.disconnect();
  console.log("all done!");
};

main();
```

Running this to make sure everything works should console log a few things. If you are running this from your own node environment, don't forget to npm i xrpl to add it to your project.

## creating wallets

Lets add some wallet creation logic, we might as well create 2 wallets to have some fun.

```
  console.log('lets fund 2 accounts...')
  const { wallet: wallet1, balance: balance1 } = await client.fundWallet()
  const { wallet: wallet2, balance: balance2 } = await client.fundWallet()
  
  console.log('wallet1', wallet1)
  console.log('wallet2', wallet2)
  
  console.log({ 
      balance1, 
      address1: wallet1.address, //wallet1.seed
      balance2, 
      address2: wallet2.address 
  })
```

At this point we should have 2 wallets with balances of 100 XRP.


