---
description: >-
  Smart contracts on Obyte are expressions that evaluate to true or false, the
  money stored on the contract can be spent only when it evaluates to true.
---

# Smart contracts

The smart contract language is declarative, meaning that it expresses **what** conditions must be met to allow movement of money, rather than **how** the decisions are made. This makes it easy to see that the implementation of the contract matches its intent, and hard to make mistakes \(which cannot be undone in distributed ledgers\).

However, the language is not as powerful as Ethereum’s Solidity, it is not Turing-complete, it doesn’t allow to code any program, rather it is a domain specific language for money on the distributed ledger. If you are looking to build more complex dApps that are as capable as Ethereum smart-contracts, then you need to use [Autonomous Agents](../autonomous-agents/).

Money on Obyte is stored on addresses. Address is just a hash \(plus checksum\) of an [address definition](reference.md), and the address definition is an expression in the Obyte smart contract language that evaluates to either `true` or `false`.

## Offering smart contracts

When you want to create a new smart contract with a user, your sequence is usually as follows:

* you ask the user to send his payment address \(it will be included in the contract\)
* you define a new contract using the user's address and your address as parties of the contract
* you pay your share to the contract
* at the same time, you send a specially formatted payment request \(different from the payment request above\) to the user to request his share. You start waiting for the user's payment
* the user views the contract definition in his wallet and agrees to pay
* you receive the payment notification and wait for it to get confirmed
* after the payment is confirmed, the contract is fully funded

We'll discuss creating and offering contracts below. These two steps are similar to what happens in the GUI wallet when a user designs a new contract.

### Creating contract definition

Create a JSON object that defines the contract:

```javascript
var arrDefinition = ['or', [
    ['and', [
        ['address', user_address],
        // conditions when user can unlock the contract
    ]],
    ['and', [
        ['address', bot_address],
        // conditions when I can unlock the contract
    ]]
]];
```

Create another object that describes the positions of your and user addresses in the above definition:

```javascript
var device = require('ocore/device.js');
var assocSignersByPath = {
    'r.0.0': {
        address: user_address,
        member_signing_path: 'r', // unused, should be always 'r'
        device_address: from_address
    },
    'r.1.0': {
        address: bot_address,
        member_signing_path: 'r', // unused, should be always 'r'
        device_address: device.getMyDeviceAddress()
    }
};
```

The keys of this object are `r` \(from "root"\) followed by indexes into arrays in the definition's `or`and `and` conditions. Since the conditions can be nested, there can be many indexes, they are separated by dot.

Then you create the smart contract address:

```javascript
var walletDefinedByAddresses = require('ocore/wallet_defined_by_addresses.js');
walletDefinedByAddresses.createNewSharedAddress(arrDefinition, assocSignersByPath, {
    ifError: function(err){
        // handle error
    },
    ifOk: function(shared_address){
        // new shared address created
    }
});
```

If the address was successfully created, it was also already automatically sent to the user, so the user's wallet will know it.

Example of vesting contracts: [Real Name Attestation](https://github.com/byteball/real-name-attestation/blob/master/modules/contract.js), [Steem Attestation](https://github.com/byteball/steem-attestation/blob/master/modules/contract.js), [Bitcointalk attestation](https://github.com/byteball/bitcointalk-attestation/blob/master/modules/contract.js).  
Examples with oracles: [Flight Delays Insurance](https://github.com/byteball/flight-delays-insurance/blob/master/offerContract.js),  [GUI wallet](https://github.com/byteball/obyte-gui-wallet/blob/master/src/js/controllers/correspondentDevice.js).  
Examples with tokens: [Condiitonal token sale](https://github.com/byteball/conditional-token-sale/blob/master/offerContractReversePayment.js), [Exchange bot](https://github.com/byteball/obyte-exchange/blob/master/book.js).

More definition examples can be seen on [smart contracts definitions page](reference.md).

### Sending a request for contract payment

To request the user to pay his share to the contract, create the below Javascript object `objPaymentRequest` which contains both the payment request and the definition of the contract, encode the object in base64, and send it over to the user:

```javascript
var payments = [
    {address: shared_address, amount: peer_amount, asset: peerAsset}
];
var definitions = {};
definitions[shared_address] = {
    definition: arrDefinition,
    signers: assocSignersByPath
};
var objPaymentRequest = {payments, definitions};
var paymentJson = JSON.stringify(objPaymentRequest);
var paymentJsonBase64 = Buffer.from(paymentJson).toString('base64');
var paymentRequestCode = 'payment:'+paymentJsonBase64;
var paymentRequestText = '[...]('+paymentRequestCode+')';

device.sendMessageToDevice(from_address, 'text', paymentRequestText);
```

The user's wallet will parse this message, display the definition of the contract in a user-readable form, and offer the user to pay the requested amount. Your payment-waiting code will be called when the payment is seen on the DAG.

## Re-sending lost smart-contracts

Since smart-contracts are private until they have been spent, users might loose them if they probably didn't backup their wallet or restored from seed words. Some attestation bots lock user's reward for months or years into smart-contracts and it would be more user-friendly if users could still restore the smart-contract even after they have lost it. Since all parties \(2 or more\) have the definition of the smart-contract in their wallet, one could ask to resend it from the other. With chat bots, this could be done like this:

```javascript
const eventBus = require('ocore/event_bus.js');

eventBus.on('text', function(from_address, text) {
	const device = require('ocore/device.js');

	if (text === 'resend') {
		const walletDefinedByAddresses = require('ocore/wallet_defined_by_addresses');
		walletDefinedByAddresses.sendToPeerAllSharedAddressesHavingUnspentOutputs(from_address, 'base', {
			ifFundedSharedAddress: function(numberOfContracts) {
				device.sendMessageToDevice(from_address, "text",
					`Found and resent ${numberOfContracts} smart contracts that have Bytes on them to your wallet.`
				);
			},
			ifNoFundedSharedAddress: function() {
				device.sendMessageToDevice(from_address, "text",
					`No smart contracts with Bytes on it were found.`
				);
			}
		});
	}
});
```

