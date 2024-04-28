#Introduction to XRPL

## getting started

Before anything else, you are going to want to setup our coding environment.

Any javascript editor will do and feel free to use your own node environement to follow along with javascript. Most of the code we will use can be run client side as well. 

If you want a hastle free setup, use replit (https://replit.com/). Accounts are free and then create a new replit with the typescript defaults or the node defaults if you would rather avoid typescript.

If you use another language, you can follow along using a script of your own, but you will have to use the appropriate SDK which may differ from xrpl.js at times.

You will often want to refer to the main API documentation for transactions here: https://xrpl.org/docs/references/protocol/transactions/types/


### faucets

While faucets are accessible programatically, you can also create a test account via this page: https://xrpl.org/resources/dev-tools/xrp-faucets/


### wallets

This tutorial makes use of the Xaman wallet, which you can download from your mobile phone at https://xumm.app. 
Once installed you can import a wallet via "family seed" (the address secret) for quicker setup. 

Most examples in this tutorial sign transactions programmatically however. 


## creating the main structure of your script 

create a new file or edit `index.ts`

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

Running this to make sure everything works should console log a few things.
If you are running this from your own node environment, don't forget to `npm i xrpl` to add it to your project.


## creating wallets 

Lets add some wallet creation logic, we might as well create 2 wallets to have some fun.

    console.log('lets fund 2 accounts...')
    const { wallet: wallet1, balance: balance1 } = await client.fundWallet()
    const { wallet: wallet2, balance: balance2 } = await client.fundWallet()

    console.log('wallet1', wallet1)

    console.log({ 
        balance1, 
        address1: wallet1.address, //wallet1.seed
        balance2, 
        address2: wallet2.address 
    })

At this point we should have 2 wallets with balances of 100 XRP.

## transfering XRP from one wallet to the other

Of course we are going to want to transfert some XRP from one account to the other, because, well that's why we're here...

    const tx:xrpl.Payment  = {
        TransactionType: "Payment",
        Account: wallet1.classicAddress,
        Destination: wallet2.classicAddress,
        Amount: xrpl.xrpToDrops("13")
    };

lets log that to make sure everything is right, and submit it to the ledger: 

    console.log('submitting the payment transaction... ', tx)

    const result = await client.submitAndWait(tx, {
        autofill: true,
        wallet: wallet1,
    }); 

    console.log(result)

When you run this part you should get the following log output: 

    {
      id: 28,
      result: {
        Account: 'rGGJ71dbSY5yF9BJUDSHsDPSKDhVGGWzpY',
        Amount: '13000000',
        DeliverMax: '13000000',
        Destination: 'rBAbErfjkwFWFerLfAHmbi3qgbRfuFWxEN',
        Fee: '12',
        Flags: 0,
        LastLedgerSequence: 232639,
        Sequence: 232617,
        SigningPubKey: 'ED13EBC7F89545435E82DC19B2C38AF5ECF39CE099C8FA647280C71CD6FA5BEF3B',
        TransactionType: 'Payment',
        TxnSignature: '66D9338B3C93D212F89BB4F6731E7F38A01052F8ED94452D7D2A9BB0B0C6130A5708390E9B6233A98A3B28A1922E57E37317609727409B3D289C456BB3250E08',
        ctid: 'C0038CAD00000001',
        date: 767648991,
        hash: 'CC8241E3C4B57ED9183D6031F4E370AC13B6CE9E2332BD7AF77C25BD6ADFA4F6',
        inLedger: 232621,
        ledger_index: 232621,
        meta: {
          AffectedNodes: [Array],
          TransactionIndex: 0,
          TransactionResult: 'tesSUCCESS',
          delivered_amount: '13000000'
        },
        validated: true
      },
      type: 'response'
    }

The most important being `TransactionResult: 'tesSUCCESS'`

You can check out the transaction actually took place on the ledger and is readable by anyone at this point. Go to https://testnet.xrpl.org and past the hash value (in my example case it would be `CC8241E3C4B57ED9183D6031F4E370AC13B6CE9E2332BD7AF77C25BD6ADFA4F6`. You should see it. Alternatively check out the wallet addresses as well (remeber you are using the *public* key).

I like to simply check the balances programmatically to make sure everything has gone according to plan, adding this would achieve that:

      console.log({
        'balance 1': await client.getBalances(wallet1.classicAddress), 
        'balance 2': await client.getBalances(wallet2.classicAddress)
      })

## Putting it all together

Here is the entire index.ts file at this point:

    import xrpl  from "xrpl" 
    
    const client = new xrpl.Client("wss://s.altnet.rippletest.net:51233")
    
    
    const main = async () => {
      console.log("lets get started...");
      await client.connect();
    
      // do something interesting here
      console.log('lets fund 2 accounts...')
      const { wallet: wallet1, balance: balance1 } = await client.fundWallet();
      const { wallet: wallet2, balance: balance2 } = await client.fundWallet();
    
      console.log('wallet1', wallet1)
    
      console.log({ 
        balance1, 
        address1: wallet1.address, //wallet1.seed
        balance2, 
        address2: wallet2.address 
      });
    
      const tx:xrpl.Payment  = {
        TransactionType: "Payment",
        Account: wallet1.classicAddress,
        Destination: wallet2.classicAddress,
        Amount: xrpl.xrpToDrops("13")
      };
    
      console.log('submitting the payment transaction... ', tx)
    
      const result = await client.submitAndWait(tx, {
        autofill: true,
        wallet: wallet1,
      }); 
    
      console.log(result)
    
      console.log({
        'balance 1': await client.getBalances(wallet1.classicAddress), 
        'balance 2': await client.getBalances(wallet2.classicAddress)
      })
    
      await client.disconnect();
      console.log("all done!");
    };
    
    main();

## extra credit

Don't you find it lame that the faucet only gives out 100 XRP? Why don't we create a function to print money to an address? 

You have all the main building blocks: 
1. you will need to create and fund new wallets with `await client.fundWallet()`
2. remember you can only transfert XRP up to the reserve, so max 90 XRP at a time for brand new accounts, unless you are brave enough to use a new transaction type? (https://xrpl.org/docs/references/protocol/transactions/types/accountdelete/) but in this case be warned you will have to wait a ledger or two before the account can be deleted!
3. the final function signature should look something like this `await printMoney({ destinationWallet, client })

I can't wait to see who writes this faster than chatGPT :)

