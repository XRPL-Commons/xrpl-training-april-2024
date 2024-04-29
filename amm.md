#Introduction to XRPL

follow along here: https://replit.com/@luc26/AMMLiveSession

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

We are going to want to save those in a more convenient way to resuse them as we move throught this tutorial.

Collect the seed value from the logs for both accounts and let's create wallets from those seeds from now on. We'll need an issuer and a receiver so here we go:

first we set the seed in code
```
const issuerSeed = "s...";
const receiverSeed = "s...";
```

then we create issuer and receiver wallets from seed in the main function as such

```
  const issuer = Wallet.fromSeed(issuerSeed);
  const receiver = Wallet.fromSeed(receiverSeed);
```

## creating tokens

### dealing with token codes

Token codes can be 3 letters or a padded hex string, here is a convenience function we will wrap our token code in to enable codes longer than 3 letters. This also needs to be done to work with our little training app so you will want to include this in you code. This could easily be done via an import for instance, or placed in the index.ts file directly. 

Here is the function code:

```
function convertStringToHexPadded(str: string): string {
  // Convert string to hexadecimal
  let hex: string = "";
  for (let i = 0; i < str.length; i++) {
    const hexChar: string = str.charCodeAt(i).toString(16);
    hex += hexChar;
  }

  // Pad with zeros to ensure it's 40 characters long
  const paddedHex: string = hex.padEnd(40, "0");
  return paddedHex.toUpperCase(); // Typically, hex is handled in uppercase
}
```

### enable rippling 

For AMMs and the whole flow to work we will need to enable rippling from the issuer account. To enable rippling we use the AccountSet transaction with the appropirate flag. Here is the function. 

```
async function enableRippling({ wallet, client }: any) {
  const accountSet: AccountSet = {
    TransactionType: "AccountSet",
    Account: wallet.address,
    SetFlag: AccountSetAsfFlags.asfDefaultRipple,
  };

  const prepared = await client.autofill(accountSet);
  const signed = wallet.sign(prepared);
  const result = await client.submitAndWait(signed.tx_blob);

  console.log(result);
  console.log("Enable rippling tx: ", result.result.hash);

  return;
}
```

and in the main function of index.ts we can now add, to trigger this
```
// enable ripling
  await enableRippling({ wallet: issuer, client });
```

### creating our issued token

To create an issued token, the receiver first needs to add a trustline to the issuer. Its a pretty straigthforward, we create a TrustSet transaction and sign it with the receiver account. 

Once the trustline is set, we send an amount from the issuer to the receiver in an amount less than the trustline maximum (500M tokens in this case)


Here is the whole file for `createToken.ts`
```
import { TrustSet, convertStringToHex, TrustSetFlags } from "xrpl";
import { Payment } from "xrpl/src/models";

async function createToken({ issuer, receiver, client, tokenCode }: any) {
  // Create the trust line to send the token
  const trustSet: TrustSet = {
    TransactionType: "TrustSet",
    Account: receiver.address,
    LimitAmount: {
      currency: tokenCode,
      issuer: issuer.address,
      value: "500000000", // 500M tokens
    },
    Flags: TrustSetFlags.tfClearNoRipple,
  };
  console.log(trustSet);

  // Receiver opening trust lines
  const preparedTrust = await client.autofill(trustSet);
  const signedTrust = receiver.sign(preparedTrust);
  const resultTrust = await client.submitAndWait(signedTrust.tx_blob);

  console.log(resultTrust);
  console.log("Trust line issuance tx result: ", resultTrust.result.hash);

  // Send the token to the receiver
  const sendPayment: Payment = {
    TransactionType: "Payment",
    Account: issuer.address,
    Destination: receiver.address,
    Amount: {
      currency: tokenCode,
      issuer: issuer.address,
      value: "200000000", // 200M tokens
    },
  };
  console.log(sendPayment);

  const preparedPayment = await client.autofill(sendPayment);
  const signedPayment = issuer.sign(preparedPayment);
  const resultPayment = await client.submitAndWait(signedPayment.tx_blob);

  console.log(resultPayment);
  console.log("Transfer issuance tx result: ", resultPayment.result.hash);


  return;
}

export default createToken;
```

We can call this from the main function of index.ts, not forgetting to wrap our token currency code with the `convertStringToHexPadded` function.

```
// create Token
  await createToken({
    issuer,
    receiver,
    client,
    tokenCode: convertStringToHexPadded("LUC"),
  });
```

We can check in the explorer that the issuer and receiver have balances in the new token at this point. 

## create an AMM Pool

Now that the receiver has tokens, we can use the receiver account to create an AMM. Note that this is the ussual architecture where the issuer account has the single purpose of issuing the token and other proprietary accounts hold the token and can create liquidity. 

For this example we are going to use Pools that have XRP as one side. 


Here is the `createAMM.ts` file:

```
import { AMMCreate, AMMDeposit, AMMDepositFlags } from "xrpl";
import { OfferCreate, OfferCreateFlags } from "xrpl";

async function createAMM({ issuer, receiver, client, tokenCode }: any) {
  console.log("create AMM", { issuer, receiver, tokenCode });
  let createAmm: AMMCreate = {
    TransactionType: "AMMCreate",
    Account: receiver.address,
    TradingFee: 600,
    Amount: {
      currency: tokenCode,
      issuer: issuer.classicAddress,
      value: "2000000", // 2M tokens
    },
    Amount2: "50000000", // 50 XRP in drops
  };
  console.log(createAmm);

  const prepared = await client.autofill(createAmm);
  const signed = receiver.sign(prepared);
  const result = await client.submitAndWait(signed.tx_blob);

  console.log(result);
  console.log("Create amm tx: ", result.result.hash);

  return;
}

export default createAMM;

```

The final main function in index.ts should now look something like this:

```
const main = async () => {
  console.log("lets get started...");
  await client.connect();

  // retrieve wallets
  const issuer = Wallet.fromSeed(issuerSeed);
  const receiver = Wallet.fromSeed(receiverSeed);

  // enable ripling
  await enableRippling({ wallet: issuer, client });

  // create Token
  await createToken({
    issuer,
    receiver,
    client,
    tokenCode: convertStringToHexPadded("LUC"),
  });

  // create AMM
  await createAMM({
    issuer,
    receiver,
    client,
    tokenCode: convertStringToHexPadded("LUC"),
  });

  await client.disconnect();

  console.log("all done!");
};
```
don't forget to import `enableRippling`, `createToken`, `createAMM` and `convertStringToHexPadded` if needed

You can now go to the training website at https://trainings.xrpl.at/ (password is training-april-2024) and interact with other pools.

You will need to setup Xaman with the receiver account that you created above. You can use the family secret import route. If you are new to Xaman don't forget to enable developer mode in the advanced settings and then switch to testnet from the home page of the xaman app. 

## Interact with your pool (swap)

Using the training app, connect and add your public address to the list. 
When you click view tokens next to an address you can see that account's availlable tokens.

You can create Trustlines for tokens you have not interacted with yet. 
You can swap XRP for other tokens where you have trustlines using the pool view's swap feature. 

You may need to print more XRP for your receiver account, that's where that print money function might come in handy. 

## Stretch goal

### Become a liquidity provider for other pools
You can become a Liquidity Provider for other pools, you will need to use the AMMDeposit function to do so (https://xrpl.org/docs/references/protocol/transactions/types/ammdeposit/)

### Withdraw from pools
You can also withdraw from pools using your LP tokens. This is done using the AMMWithdraw transaction (https://xrpl.org/docs/references/protocol/transactions/types/ammwithdraw/)


Who got the most tokens?


