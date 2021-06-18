# Contracts with arbitration

Contracts with arbitration are peer-to-peer free-form textual contracts coupled with simple smart contracts that safely lock the money for the duration of the contract and allow disputes to be resolved by mutually agreed arbiters. In other words, a contract with arbitration is a [prosaic contract](prosaic-contracts.md) plus a smart contract that guarantees that the parties faithfully follow the contract.

If you are developing a bot that makes contract offers, here is an outline of the contract's lifecycle:

* Your bot \(offerer\) receives a user's address in chat from a user.
* The bot then sends a contract with arbitration offer to the user. In the offer, it sets the user's address as the other party of the contract, amount of the contract, and an arbiter's address of your choosing. You can also choose the role for your bot in this contract \(buyer or seller\).
* The user reviews the offer in their wallet and accepts the contract.
* Your bot receives the user's acceptance, generates a new shared address with smart contract definition, and initiates signing and posting of a special `data` unit with the hash of the agreed contract text. The unit is to be signed by the shared address \(shared between the bot and the user\) and each party's signature means that they agree with the contract's text. Your bot signs first, then waits for the user to sign the transaction.
* After the unit has been posted, the buyer side of the contract should pay to this contract's address the full amount that was specified in the contract.
* As soon as the funds are locked on the contract, the seller of the goods/services starts fulfilling their obligations.
* When all the work is done and the buyer is satisfied with the result, they can release the funds from the contract sending them to the seller's address. Also at any time the seller can refund the full amount back to the buyer. The smart contract ensures that in the absence of any arbiter decision each party can send the contract's funds only to the other party: buyer to the seller to release the funds after the work is done, or seller to the buyer to refund. Either outcome completes the contract.
* In case of a disagreement, the user can open a dispute and invoke the arbiter by clicking the corresponding button in their wallet, or your bot can call an API method to open a dispute. The plaintiff has to pay for the arbiter's service before the arbiter starts looking into the case. After the contract status becomes `in_dispute`, both parties have to wait for the arbiter to post a resolution unit. After the arbiter has studied the contract and the evidence provided by the parties, they make a decision and post the resolution unit. Then, the winning side can claim the funds from the contract.

Note that in order to open a dispute from the bot, your bot should implement the corresponding API \(see below\). After you open a dispute, the arbiter will set a fee for their service and the arbstore will send a payment request to your bot. Your bot should be able to handle such requests and decide about paying them, either autonomously, or forward them to a human operator.

All the API methods available for your bot are located in [ocore/arbiter\_contract.js](https://github.com/byteball/ocore/blob/master/arbiter_contract.js) file.

#### Example

There is an example script for a part of the API functionality in [tools/arbiter\_contract\_example.js](https://github.com/byteball/headless-obyte/blob/master/tools/arbiter_contract_example.js) inside `headless-wallet` repository, you can start by modifying it to your needs:

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

Basically, all you need to do is calling the exported methods of `arbiter_contract.js` module and reacting to `arbiter_contract_*` events on the event bus.

#### API reference:

1. `createAndSend(contractObj, callback(contract){})`

   This method creates a contract from the provided object and sends it to the peer. `contractObj` should contain all the fields needed to assemble a new contract offer. The function is then sends the offer to the `contractObj.peer_device_address` via chat. The `contract` received in the callback function has `hash` field, it is the contract's hash which acts as the primary key when referencing contracts.

   `contractObj` required fields: 

   1. `title` - contract title, `string`
   2. `text` - contract text, `string`
   3. `arbiter_address` - the address of the picked arbiter, `string`
   4. `amount` - amount in `asset` to be paid to the seller, `int`
   5. `asset` - asset in which the payment should be done, can be `null` or `"base"` for Bytes, or any other asset ID, `string`
   6. `peer_address` - address of the other side of the contract, `string`
   7. `my_address` - your address, `string`
   8. `me_is_payer` - flag specifying if you are the buyer or seller, `bool`
   9. `peer_device_address` - the device address of the other side of the contract, `string`
   10. `ttl` - Time-To-Live, in hours, till which the contract can be accepted, then it expires, `float`
   11. `cosigners` - array of your wallet cosigners that would be used when posting contract signature unit, leave empty if not multi-sig or when you want to use all of the cosigners, `array of device addresses`
   12. `my_contact_info` - free-form text with your contact info so the peer and arbiter can contact you, it is important to note that if you are sending contract offers from a bot, then arbiter can only pair with your bot and not you, therefore you should put your own wallet's pairing code here or other contacts by which you \(as a human\) can be reached,`string`

   `contract` object supplied to your callback will mostly be identical to the contractObj that you provided to this function, but with some additional fields populated, such as `hash, creation_date, my_pairing_code.`  

2. `createSharedAddressAndPostUnit(contract_hash, walletInstance, callback(error, contract){})`

   This method is used to react to the peer accepting your offer. It creates a new shared address with smart contract definition and posts the unit into the DAG. `walletInstance` is a reference to wallet that has `sendMultiPayment()` method with an appropriate signer inside. For bots, it is usually the main `headlessWallet` module. If the `error` argument for the callback is `null`, it means that the unit was posted successfully, the `contract` object is returned with the updated fields.  

3. `pay(contract_hash, walletInstance, arrSigningDeviceAddresses, callback(error, contract, unit){})`

   This method pays to the contract if you are the buyer side. If you use a multi-signature wallet, then fill the `arrSigningDeviceAddresses` array with signing device addresses, otherwise provide an empty array. The `unit` argument in the callback function is the unit ID in case the payment succeeded.  

4. `complete(contract_hash, walletInstance, arrSigningDeviceAddresses, callback(error, contract, unit){})`

   This method sends the funds locked on the contract to the peer. Depending on your role, this action performs either a successful completion \(you are a buyer\), or a refund \(you are the seller\).  

5. `respond(contract_hash, status, signedMessageBase64, signer, callback(error, contract){})`

   If you allow users to make offers, use this function after receiving a contract offer, usually as a reaction to the event `arbiter_contract_offer`. To get a better idea how to construct `signedMessageBase64` and `signer` arguments for this function, you can take a look at an example of how GUI wallet is invoking it: [https://github.com/byteball/obyte-gui-wallet/blob/95d3354de9f9156fd06ccc63c0a4cac8c94f2510/src/js/services/correspondentService.js\#L669](https://github.com/byteball/obyte-gui-wallet/blob/95d3354de9f9156fd06ccc63c0a4cac8c94f2510/src/js/services/correspondentService.js#L669)  

6. `openDispute(contract_hash, callback(error, response, contract){})`

   This function raises a dispute by making an HTTPS request to the contract arbiter's current ArbStore web address. Callback is provided with an error, if any, http response and the updated contract object.  

7. `appeal(contract_hash, callback(error, response, contract){})`

   This method is similar to the previous one, but is used when you are not satisfied with the arbiter's decision. Also makes an HTTPS request to the ArbStore, but this time calls to ArbStore moderator, who can penalize the arbiter, in case they find the arbiter's decision / actions inappropriate.  

8. `getByHash(contract_hash, callback(contract){})` / `getBySharedAddress(address, callback(contract){})` / `getAllBy*()`

   These are all similar functions to retrieve one or many contracts by specific criteria. To get a detailed overview of all the methods you can look at exported methods of `arbiter_contract.js` module.

#### Event reference

Almost every event is supplied with additional arguments that you can use in your event handler functions. For examples, see the code snippet above.

1. `arbiter_contract_offer : (contract_hash)` - this event is fired when core received new contract with arbitration offer from any correspondent.
2. `arbiter_contract_response_received : (contract)` - fired when user responds to your contract offer. To get their response, check the `status` field of the contract, it should be either `"accepted"` or `"declined"`.
3. `arbiter_contract_update : (contract, field, value, unit, winner, isPrivate)` - this is the most common event that you will react to. It is raised in many situations, like:
   * changed status of a contract: `field = "status"` \(contract was `revoked`, or put `in_dispute`, `in_appeal`, was `paid`, `completed`, etc.\). This is the full list of the contract statuses: `'pending', 'revoked', 'accepted', 'signed', 'declined', 'paid', 'in_dispute', 'dispute_resolved', 'in_appeal', 'appeal_approved', 'appeal_declined', 'cancelled', 'completed'`
   * unit with contract hash was posted: `field = "unit"`

