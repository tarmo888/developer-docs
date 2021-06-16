# Contracts with arbitration

Contracts with arbitration are the feature of Obyte platform to allow users to enter into aggreements via signing peer-to-peer free-form textual contracts with added benefits in the form of third party arbiters. They are similar to prosaic contracts and allow you to specify the exact amount of money to be paid to the seller of services, along with picking an arbiter, who can resolve any disputes in case any is raised.

A scheme of making contract offers from your bot can look like these:

* Your bot \(offerer\) receives an address in chat from the user.
* Bot then sends a contract with arbitration offer to the user, setting user's address as a second party of the contract, amount of the contract and some specific arbiter's address which you like. You can also choose the role for your bot in this contract \(buyer/seller\).
* User accepts the contract, your bot handles this response, generates new shared address with special definition and posts a special 'data' unit with the agreeg contract text hash, which is signed by the shared address. As the address is shared between your bot and the user, you have to wait until the user signs the transaction.
* After the unit was posted, the buyer side of the contract should pay to this contract's address the full amount which was specified during contract offer creation.
* As soon as the funds are locked on the contract, the seller of the services can fulfill his obligations.
* When all the work is done and buyer is satisfied with the result, they can release the funds from the contract, sending them to the seller's address. Also at any time the seller can refund the money back to the buyer. Either way completes the contract.
* In case the call to arbiter is needed, the user can click the button in his waller directly from the contract or your bot can call the API method for it. The plaintiff have to pay for arbiter's service first before they can resolve the dispute. After the contract is disputed, both parties have to wait for an arbiter to post a resolution unit, after the resolution was posted, the winning side can claim the funds from the contract.

All the API methods available for your bot are located in [ocore/arbiter\_contract.js](https://github.com/byteball/ocore/blob/master/arbiter_contract.js) file.

#### Example

There is an example script for a part of the API functionality in [tools/arbiter\_contract\_example.js](https://github.com/byteball/headless-obyte/blob/master/tools/arbiter_contract_example.js) inside `headless-wallet` repository, you can start by modifiying it to your needs:

```javascript
"use strict";
const headlessWallet = require('../start.js');
const eventBus = require('ocore/event_bus.js');
const arbiter_contract = require('ocore/arbiter_contract.js');
const device = require('ocore/device.js');
const validationUtils = require('ocore/validation_utils');

function onReady() {
	let contract_text = "Bill pays Tom $20, if Tom sends Bill a pair of his Air Jordans.";
	let contract_title = "Air Jordans Purchase from Tom";
	let amount = 10000; // in bytes, min 10000
	let asset = null;
	let ttl = 24; //hours
	let arbiter_address = "VYDZPZABPIYNNHCPQKTMIKAEBHWEW3SQ";
	let my_contacts = `Email me at: bill@example.com`;
	let me_is_payer = true;

	let my_address;
	headlessWallet.readFirstAddress(address => {
		my_address = address;
	});

	eventBus.on('paired', from_address => {
		device.sendMessageToDevice(from_address, 'text', `My address: ${my_address}, now send me your's.`);
	});

	/* ================ OFFER ================ */
	eventBus.on('text', (from_address, text) => {
		text = text.trim();
		if (!validationUtils.isValidAddress(text))
			return device.sendMessageToDevice(from_address, 'text', `does not look like an address`);
		let contract = {
			title: contract_title,
			text: contract_text,
			arbiter_address: arbiter_address,
			amount: amount,
			asset: asset,
			peer_address: text,
			my_address: my_address,
			me_is_payer: me_is_payer,
			peer_device_address: from_address,
			ttl: ttl,
			cosigners: [],
			my_contact_info: my_contacts
		};

		arbiter_contract.createAndSend(contract, contract => {
			console.log('contract offer sent', contract);
		});
	});

	/* ================ OFFER ACCEPTED ================ */
	eventBus.on("arbiter_contract_response_received", contract => {
		if (contract.status != 'accepted') {
			console.warn('contract declined');
			return;
		}
		arbiter_contract.createSharedAddressAndPostUnit(contract.hash, headlessWallet, (err, contract) => {
			if (err)
				throw err;
			console.log('Unit with contract hash was posted into DAG\nhttps://explorer.obyte.org/#' + contract.unit);

	/* ================ PAY TO THE CONTRACT ================ */
			if (contract.me_is_payer) {
				arbiter_contract.pay(contract.hash, headlessWallet, [],	(err, contract, unit) => {
					if (err)
						throw err;
					console.log('Unit with contract payment was posted into DAG\nhttps://explorer.obyte.org/#' + unit);

					setTimeout(() => {completeContract(contract)}, 3 * 1000); // complete the contract in 3 seconds
				});
			}
		});
	});

	/* ================ CONTRACT FULFILLED - UNLOCK FUNDS ================ */
	function completeContract(contract) {
		arbiter_contract.complete(contract.hash, headlessWallet, [], (err, contract, unit) => {
			if (err)
				throw err;
			console.log(`Contract completed. Funds locked on contract with hash ${contract.hash} were sent to peer, unit: https://explorer.obyte.org/#${unit}`);
		});
	}

	/* ================ CONTRACT EVENT HANDLERS ================ */
	eventBus.on("arbiter_contract_update", (contract, field, value, unit) => {
		if (field === "status" && value === "paid") {
			// do something usefull here
			console.log(`Contract was paid, unit: https://explorer.obyte.org/#${unit}`);
		}
	});
};
eventBus.once('headless_wallet_ready', onReady);
```

Basically, all you need to do is to call exported methods of `arbiter_contract.js` module and react to `arbiter_contract_*` events on event bus.

#### API reference:

1. `createAndSend(contractObj, callback(contract){})`

   This method creates contract from provided object and sends it to peer. `contractObj` should contain all the fields needed to assemble a new contract offer. The offer is then sent to the `contractObj.peer_device_address` via chat. The contract received in callback function has `hash` field, it is the contract's hash which acts as a primary key when referencing contracts.

   `contractObj` required fields: 

   1. `title` - contract title, `string`
   2. `text` - contract text, `string`
   3. `arbiter_address` - the address of the picked arbiter, `string`
   4. `amount` - amount to be paid to the seller, `int`
   5. `asset` - asset in which payment with the amount specified should be done, can be `null, "base",` or any other `asset ID`, `string`
   6. `peer_address` - address of the other side of the contract, `string`
   7. `my_address` - your address, `string`
   8. `me_is_payer` - flag specifying if you are the buyer or seller, `bool`
   9. `peer_device_address` - the device address of the other side of the contract, `string`
   10. `ttl` - Time-To-Live, in hours, till which the contract can be accepted, before it expires, `float`
   11. `cosigners` - array of your wallet cosigners that would be used when posting contract signature unit, leave empty if not multi-sig or when you want to use all of the cosigners, `array of device addresses`
   12. `my_contact_info` - free form text with your contact info so the peer and arbiter can contact you, it is important to note that if you sending contract offers from a bot, then arbiter can only pair with your bot and not you, therefore you should put your own wallet's pairing code in here or other contacts by which you \(as a human\) are accessible,`string`

   `contract` object supplied to your callback will mostly be identical to the contractObj that you provided to this function, but with some additional fields populated, such as `hash, creation_date, my_pairing_code.`  

2. `createSharedAddressAndPostUnit(contract_hash, walletInstance, callback(error, contract){})`

   This method is used as a reaction when peer accepts the contract. It creates new shared address with special definition and posts the unit into the DAG. `walletInstance` is a reference to wallet that has `sendMultiPayment()` method with appropriate signer inside. For bots, it usually the main `headlessWallet` module. If the `error` argument for the callback is null, it means the unit was posted, the `contract` object returned with updated fields.  

3. `pay(contract_hash, walletInstance, arrSigningDeviceAddresses, callback(error, contract, unit){})`

   This method pays to the contract, if you are the buyer side. If you have multi-signature wallet, then fill the `arrSigningDeviceAddresses` array with signing device addresses, otherwise provide an empty array. The `unit` argument in callback function is unit ID in case payment succeded.  

4. `complete(contract_hash, walletInstance, arrSigningDeviceAddresses, callback(error, contract, unit){})`

   This method sends funds locked on the contract to the other peer. Depending on your role, this action either considered as a succesfull completion \(you are a buyer\), or a refund \(you are the seller\).  

5. `respond(contract_hash, status, signedMessageBase64, signer, callback(error, contract){})`

   Respond is used upon receiving contract offer, usually as a reaction to event `arbiter_contract_offer`. To get a better understanding of how to construct `signedMessageBase64` and `signer` arguments for this function, you can take a look at an example of how GUI wallet is invoking it: [https://github.com/byteball/obyte-gui-wallet/blob/95d3354de9f9156fd06ccc63c0a4cac8c94f2510/src/js/services/correspondentService.js\#L669](https://github.com/byteball/obyte-gui-wallet/blob/95d3354de9f9156fd06ccc63c0a4cac8c94f2510/src/js/services/correspondentService.js#L669)  

6. `openDispute(contract_hash, callback(error, response, contract){})`

   This function raises a dispute by making an HTTPS request to contract arbiter's current ArbStore web address. Callback is provided with error, if any, http response and updated contract object.  

7. `appeal(contract_hash, callback(error, response, contract){})`

   This method is similar to previous one, but is used when you are not satisfied with the outcome of arbiter actions. Also makes an HTTPS request to the ArbStore, but this time calls to ArbStore moderator, who can penalize the arbiter, in case their decision / actions were inappropriate.  

8. `getByHash(contract_hash, callback(contract){})` / `getBySharedAddress(address, callback(contract){})` / `getAllBy*()`

   These are all similar functions to retrieve one or many contracts with some specific criterias. To get a detailed overview of all methods you can look at exported methods of `arbiter_contract.js` module.

#### Event reference

Almost every event is supplied with additional arguments that you can use in your event handler functions.

1. `arbiter_contract_offer : (contract_hash)` - this event is fired when core received new contract with arbitration offer from any correspondent.
2. `arbiter_contract_response_received : (contract)` - fired when user responds to your contract offer. To get their response, check the `status` field of the contract, it should be either `"accepted"` or `"declined"`.
3. `arbiter_contract_update : (contract, field, value, unit, winner, isPrivate)` - this is the most common event that you will react to. It is raised in many situations, like:
   * changed status of a contract \(contract was `revoked`, or put `in_dispute`, `in_appeal`, was `paid`, `completed`, etc.\). Full list of contract statuses possible: `'pending', 'revoked', 'accepted', 'signed', 'declined', 'paid', 'in_dispute', 'dispute_resolved', 'in_appeal', 'appeal_approved', 'appeal_declined', 'cancelled', 'completed'`
   * unit with contract hash was posted, then `field === "unit"`

