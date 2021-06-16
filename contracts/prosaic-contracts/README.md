# Prosaic contracts

Prosaic contracts are the feature of Obyte platform that helps people to make an agreement on, basically, anything, as text of contracts is a simple text. The benefit of using Obyte prosaic contracts is that after both sides of the contract sign it, a unit, which in fact is a public proof of contract agreement, is posted into DAG, which contains a hash of contract text, while being signed by both peers of the contract. So anyone can check the fact of agreement on the exact same text by those two contract peers. The contract text itself is not revealed but stored in peers wallets.

To offer a prosaic contract from your bot, you have to follow these steps:

* user pairs with your bot.
* you ask the user to send his payment address \(it will be included in the contract\) in the chat.
* you [send the user the private profile](../../private-profiles.md#sending-private-profile) of the other side of the contract.
* you create a new prosaic contract using the user's address and your address as parties of the contract \(also defining contract's title, text and how long the offer is valid\).
* you send specially formatted chat message with the contract to the user and wait for the response on it \('accept' or 'decline'\).
* the user views the contract in his wallet and agrees or declines it.
* on receiving 'accepted' response, you generate new shared address with special definition \('and' between two addresses\).
* then you top up this shared address to compensate fees.
* you initiate posting a 'data' message from this shared address, containing contract text hash. User will then be asked to sign this transaction from the same shared address \(as he is a part of its definition\).
* after getting the user's signature for the 'data' unit, you post it into DAG. Mission accomplished.

We have a special wrapper API for all of these steps in `ocore/prosaic_contract_api.js`, which in turn is calling `ocore/prosaic_contract.js` methods. For simplicity, here is an example of using `*_api.js`, you can check it's code to get a better understanding of what's going on.

{% code title="start.js" %}
```javascript
"use strict";
const headlessWallet = require('headless-obyte');
const eventBus = require('ocore/event_bus.js');
const prosaic_contract = require('headless-obyte/prosaic_contract_api.js');

function onReady() {
    prosaic_contract.listenForPendingContracts(headlessWallet.signWithLocalPrivateKey);

    let contract_text = "I pay Tom $20, if he sends me a pair of his Air Jordans.";
    let contract_title = "Air Jordans Purchase from Tom";
    let contract_is_valid_until = 24; //hours

    headlessWallet.readFirstAddress(my_address => {
        let peer_address = "62SU7724ME5MWB7VEP64VURYZHDRSDWN";
        let peer_device_address = "0PUQK5HOEKS4NONGL2ITJI3HMJ76P3LKI";
        prosaic_contract.offer(contract_title,
                                contract_text,
                                my_address,
                                peer_address,
                                peer_device_address,
                                contract_is_valid_until,
                                [],
                                headlessWallet.signWithLocalPrivateKey, {
            onOfferCreated: function(objContract) {
                console.log("on offer created", objContract);
            },
            onResponseReceived: function(accepted) {
                console.log("on response received", accepted);
            },
            onSigned: function(objContract) {
                console.log("on signed", objContract);
            },
            onError: function(error) {
                console.log("on error", error);
            }
        });
    });
};
eventBus.once('headless_wallet_ready', onReady);
```
{% endcode %}

`callbacks` object consists of four optional callback functions, which called in following sequence:

* `onOfferCreated` is called right after the contract was created and sent to peer. Always called.
* `onResponseReceived` is called when peer respond to the contract, supplying the response in `accepted` bool argument.
* `onSigned` indicates that the contract was successfully signed and unit with contract's hash was posted into DAG.
* `onError` can only happen when handling the response, if your bot did receive any response, either `onError` or `onSigned` will be called.

The core module for prosaic contracts is [ocore/prosaic\_contract.js](https://github.com/byteball/ocore/blob/master/prosaic_contract.js). You can directly use it's exported methods, so check the content of it first.

