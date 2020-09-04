---
description: >-
  Event bus provides a way to subscribe to events from any of the services
  running.
---

# Events list

```javascript
const eventBus = require('ocore/event_bus.js');
```

### Wallet is ready \(only headless wallet\)

Headless wallet is needed for bots that hold private keys and can send transactions. This event is emitted when passphrase has been entered by bot admin and single wallet is loaded. Before that event, bot outputs device address, device pubkey and pairing key.

```javascript
eventBus.on('headless_wallet_ready', () => {

});
```

### Start database update

This event is emitted when bot executes series database queries to upgrade the database to new version.

```javascript
eventBus.on('started_db_upgrade', () => {

});
```

### End of database update

This event is emitted when bot finishes database upgrade.

```javascript
eventBus.on('finished_db_upgrade', () => {

});
```

### Connection to the hub via ws is established

```javascript
eventBus.on('connected', () => {

});
```

### Connection to the hub is established and an error is possible

The `url` parameter is protocol + domain + path \(for example: `wss://obyte.org/bb`\).

```javascript
eventBus.on('open-' + url, (error) => {

});
```

### Connected to ws

```javascript
eventBus.on('connected_to_source', (new_ws) => {

});
```

### Not a fatal error

```javascript
eventBus.on('nonfatal_error', (msg, Error) => {

});
```

### Unit validation status

```javascript
eventBus.on("validated-" + unit, (bValid) => {

});
```

### New joint

```javascript
eventBus.on('new_joint', (objJoint) => {

});
```

### Main chain index has become stable

```javascript
eventBus.on('mci_became_stable', (mci) => {

});
```

### New unstable transactions

```javascript
eventBus.on('new_my_transactions', (arrNewUnits) => {

});
```

### Stable transactions

```javascript
eventBus.on('my_transactions_became_stable', (arrUnits) => {

});
```

### Changes with addresses leading to possible new transactions

```javascript
eventBus.on('maybe_new_transactions', () => {

});
```

### New unstable unit

```javascript
eventBus.on("new_my_unit-" + unit, (objJoint) => {

});
```

### New private payment

```javascript
eventBus.on("received_payment", (payer_device_address, assocAmountsByAsset, asset, message_counter, bToSharedAddress) => {

});
```

### Stable unit

```javascript
eventBus.on('my_stable-' + unit , () => {

});
```

### Catching up done

```javascript
eventBus.on('catching_up_done' , () => {

});
```

### Catching up started

```javascript
eventBus.on('catching_up_started' , () => {

});
```

### Catchup next hash tree

```javascript
eventBus.on('catchup_next_hash_tree' , () => {

});
```

### New direct private chains

```javascript
eventBus.on('new_direct_private_chains' , (arrChains) => {

});
```

### Unhandled private payments left

```javascript
eventBus.on('unhandled_private_payments_left' , (rowsLength) => {

});
```

### Version of the connected user

```javascript
eventBus.on('peer_version', (ws, body) => {

});
```

### New wallet version available

This event is emitted when Hub sends message about new wallet version.

```javascript
eventBus.on('new_version', (ws, body) => {

});
```

### Received the project number for notifications on the phone

```javascript
eventBus.on('receivedPushProjectNumber', (ws, body) => {

});
```

### Client logged in

```javascript
eventBus.on('client_logged_in', (ws) => {

});
```

### Message from the hub

```javascript
eventBus.on("message_from_hub", (ws, subject, body) => {

});
```

### **Message for light client**

```javascript
eventBus.on("message_for_light", (ws, subject, body) => {

});
```

### Exchange rates are updated

This event is emitted when wallet updated rates that they got from Hub.

```javascript
const network = require('ocore/network.js');
eventBus.on("rates_updated", () => {
    console.log(JSON.stringify(network.exchangeRates, null, 2));
});
```

### Message received by hub

```javascript
eventBus.on('peer_sent_new_message', (ws, objDeviceMessage) => {

});
```

### Notifications are enabled

```javascript
eventBus.on("enableNotification", (device_address, params) => {

});
```

### Notifications are disabled

```javascript
eventBus.on("disableNotification", (device_address, params) => {

});
```

### Create new wallet

```javascript
eventBus.on("create_new_wallet", (walletId, wallet_definition_template, arrDeviceAddresses, wallet_name, other_cosigners, is_single_address) => {

});
```

### Wallet created and signed

```javascript
eventBus.on('wallet_completed', (walletId) => {

});
```

### Wallet was rejected by the other side

```javascript
eventBus.on('wallet_declined', (wallet, rejector_device_address) => {

});
```

### Wallet has been confirmed by the other party

```javascript
eventBus.on('wallet_approved', (wallet, device_address) => {

});
```

### Created new address

```javascript
eventBus.on('new_wallet_address', (address) => {

});
```

### Created new address

```javascript
eventBus.on("new_address-" + address, () => {

});
```

### Unit saved

```javascript
eventBus.on('saved_unit-' + unit, (objJoint) => {

});
```

### Attempt to pair us with another correspondent

This event is emitted when there is a pairing attempt, this enables bot to decide with the code if the pairing code is valid or not. If you would like to accept any pairing code then [there is easier solution](configuration.md#accepting-incoming-connections).

```javascript
eventBus.on("pairing_attempt", (from_address, pairing_secret) => {
    if (pairing_secret == "SOME_SECRET") {
        eventBus.emit("paired", from_address, pairing_secret);
    }
});
```

### Pairing was successful

This event is emitted on successful pairing \(also when user removed and re-added the bot\).

```javascript
eventBus.on("paired", (from_address, pairing_secret) => {
    let device = require('ocore/device.js');
    // say hi to anyone who pair with bot
    device.sendMessageToDevice(from_address, 'text', 'Hi!');
});
```

### **P**airing was successful using pairing secret

```javascript
eventBus.on('paired_by_secret-' + body, (from_address) => {

});
```

### The paired device has been removed

```javascript
eventBus.on("removed_paired_device", (from_address) => {

});
```

### History update started

```javascript
eventBus.on('refresh_light_started', () => {

});
```

### History update finished

```javascript
eventBus.on('refresh_light_done', () => {

});
```

### A new message string type message has been received

This event is emitted when text message is received by bot.

```javascript
eventBus.on("text", (from_address, body, message_counter) => {
    let device = require('ocore/device.js');
    // echo back the same message
    device.sendMessageToDevice(from_address, 'text', 'ECHO: '+ body);
});
```

### A new message object type message has been received

This event is emitted when object message is received by bot.

```javascript
eventBus.on("object", (from_address, body, message_counter) => {
    let device = require('ocore/device.js');
    // echo back the same object back
    device.sendMessageToDevice(from_address, 'object', body);
});
```

###  Change in permission to store message history

```javascript
eventBus.on("chat_recording_pref", (from_address, body, message_counter) => {

});
```

### Created a new shared address

```javascript
eventBus.on("create_new_shared_address", (address_definition_template, assocMemberDeviceAddressesBySigningPaths) => {

});
```

### Signing request

```javascript
eventBus.on("signing_request", (objAddress, address, objUnit, assocPrivatePayloads, from_address, signing_path) => {

});
```

### Signed event

```javascript
eventBus.on("signature-" + device_address + "-" + address + "-" + signing_path + "-" + buf_to_sign.toString("base64"), (sig) => {

});
```

### Refuse to sign

```javascript
eventBus.on('refused_to_sign', (device_address) => {

});
```

### Confirmed on another device

```javascript
eventBus.on('confirm_on_other_devices', () => {

});
```

### All private payments are processed

```javascript
eventBus.on('all_private_payments_handled', (from_address) => {

});
```

### All private payments are processed in a unit

```javascript
eventBus.on('all_private_payments_handled-' + first_chain_unit, () => {

});
```

### Event for custom Request

You can add your own communication protocol on top of the Obyte one. See Request example [there](websocket-api/request.md#custom-request).

```javascript
const network = require('ocore/network.js');

eventBus.on('custom_request', function(ws, params, tag) {
    var response = 'put response here';
    return network.sendResponse(ws, tag, response);
}
```

### Event for custom JustSaying

You can add your own communication protocol on top of the Obyte one. See JustSaying example [there](websocket-api/justsaying.md#custom-justsaying).

```javascript
eventBus.on('custom_justsaying', function(ws, content) {

};
```

### Database is read and ready to work \(only Cordova\)

```javascript
eventEmitter.once('ready', () => {
   console.log("db is now ready");
});
```

### Error sending message via ws

```javascript
ws.on('error', (error) => {

});
```

