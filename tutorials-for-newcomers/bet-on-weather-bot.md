# Bet on weather bot

Hello, everybody. Today we will learn how to create our first contract. We will do this on the example of a bot that accepts a bet on the weather.

We will need [bot-example](https://github.com/byteball/bot-example). Let’s install it.

```bash
git clone https://github.com/byteball/bot-example
cd bot-example
npm install
cp .env.testnet .env
```

First, let’s create variables.

```javascript
const request = require('request');
const correspondents = require('./correspondents');

let steps = {}; // store user steps
let assocDeviceAddressToCurrentCityAndTemp = {}; 
let assocDeviceAddressToAddress = {};
let bets = [];
let KEY_API = '50ace22603c84daba1780432180111'; // apixu.com
let my_address;
```

Save our address for use in the contract

```javascript
headlessWallet.readSingleWallet(walletId => {
  db.query("SELECT address FROM my_addresses WHERE wallet=?", [walletId], function (rows) {
     if (rows.length === 0)
        throw Error("no addresses");
     my_address = rows[0].address;
  });
});
```

Let’s add a text handler and analyze what happens:

```javascript
eventBus.on('text', (from_address, text) => {
     text = text.trim().toLowerCase();
     if (!steps[from_address]) steps[from_address] = 'start';
     let step = steps[from_address];
     const device = require('ocore/device.js');
     if (validationUtils.isValidAddress(text.toUpperCase())) { // If address sent to us - save it.
        assocDeviceAddressToAddress[from_address] = text.toUpperCase();
        return device.sendMessageToDevice(from_address, 'text', 'I saved your address. Send me the name of the city');
     } else if (!assocDeviceAddressToAddress[from_address]) { // If we do not know the address - asking it to send for us.
        return device.sendMessageToDevice(from_address, 'text', 'Please send me your byteball address(... > Insert my address)');
     } else if (step === 'city' && (text === 'more' || text === 'less')) { // Create a contract and ask to pay for it
        let time = Date.now() + 20 * 60 * 1000;
        let name = assocDeviceAddressToCurrentCityAndTemp[from_address].city + '_' + time;
        let operator = text === 'more' ? '>=' : '<=';
        let value = assocDeviceAddressToCurrentCityAndTemp[from_address].temp;
        createContract(my_address, assocDeviceAddressToAddress[from_address], 2001, 2000, from_address, name, operator, value, time,
           (err, paymentRequestText, shared_address, timeout) => {
              if (err) throw err;
              findOracleAndSendMessage(assocDeviceAddressToCurrentCityAndTemp[from_address].city + ':' + time, () => {
                 bets.push({
                    myAmount: 2001,
                    peerAmount: 2000,
                    myAddress: my_address,
                    peerAddress: assocDeviceAddressToAddress[from_address],
                    sharedAddress: shared_address,
                    peerDeviceAddress: from_address,
                    name,
                    operator,
                    value,
                    timeout
                 });
                 device.sendMessageToDevice(from_address, 'text', paymentRequestText);
                 setTimeout(() => { // If the payment was not made for 10 minutes-we take the money
                    headlessWallet.sendAllBytesFromAddress(shared_address, my_address, from_address, (err, unit) => {
                       if (!err) console.error('unit:: ', unit);
                    });
                 }, 15 * 60 * 1000);
              });
           });
     } else if (step === 'start') { // Check the weather and specify what rate the user wants to make
        switch (text) {
           case 'berlin':
           case 'moscow':
           case 'helsinki':
           case 'washington':
              request('https://api.apixu.com/v1/current.json?key=' + KEY_API + '&q=' + text, function (error, response, body) {
                 if (error) {
                    console.error(error);
                    device.sendMessageToDevice(from_address, 'text', 'An error occurred. Try again later.');
                 } else {
                    let result = JSON.parse(body);
                    let temp = result.current.temp_c;
                    device.sendMessageToDevice(from_address, 'text', 'Want to bet that in an hour the weather in ' + text + ' will be\n[more ' + temp + 'c](command:more) or [less ' + temp + 'c](command:less)?');
                    steps[from_address] = 'city';
                    assocDeviceAddressToCurrentCityAndTemp[from_address] = {city: text, temp};
                 }
              });
              break;
           default:
              device.sendMessageToDevice(from_address, 'text', "City not support");
              break;
        }
     } else {
        device.sendMessageToDevice(from_address, 'text', "unknown command");
     }
  });
});
```

Now let’s talk about contract creation.  
We need a separate function

```javascript
function createContract(myAddress, peerAddress, myAmount, peerAmount, peerDeviceAddress, name, operator, value, time, cb) {
  const device = require('ocore/device');
  let timeout = Date.now() + 10 * 60 * 1000;
  let inverseOperator = operator === '>=' ? '<' : '>';
  let arrSeenConditionPeer = ['seen', {
     what: 'output',
     address: 'this address',
     asset: 'base',
     amount: peerAmount
  }];
  let arrSeenConditionMyInput = ['seen', {
     what: 'input',
     address: 'this address',
     asset: 'base'
  }];
  let arrDefinition = ['or', [
     ['or', [
        ['and', [
           ['address', peerAddress],
           arrSeenConditionPeer,
           arrSeenConditionMyInput
        ]],
        ['or', [
           ['and', [
              ['address', peerAddress],
              ['in data feed', [[conf.oracle_address], name, operator, value]],
           ]],
           ['and', [
              ['address', myAddress],
              ['in data feed', [[conf.oracle_address], name, inverseOperator, value]]
           ]]
        ]],
     ]],
     ['and', [
        ['address', myAddress],
        ['not', arrSeenConditionPeer],
        ['in data feed', [[conf.TIMESTAMPER_ADDRESS], 'timestamp', '>', timeout]]
     ]]
  ]];
  let assocSignersByPath = {
     'r.0.0.0': {
        address: peerAddress,
        member_signing_path: 'r',
        device_address: peerDeviceAddress
     }, 'r.0.1.0.0': {
        address: peerAddress,
        member_signing_path: 'r',
        device_address: peerDeviceAddress
     },
     'r.1.0': {
        address: myAddress,
        member_signing_path: 'r',
        device_address: device.getMyDeviceAddress()
     },
     'r.0.1.1.0': {
        address: myAddress,
        member_signing_path: 'r',
        device_address: device.getMyDeviceAddress()
     }
  };
 
  let walletDefinedByAddresses = require('ocore/wallet_defined_by_addresses.js');
  walletDefinedByAddresses.createNewSharedAddress(arrDefinition, assocSignersByPath, {
     ifError: (err) => {
        cb(err);
     },
     ifOk: (shared_address) => {
        headlessWallet.issueChangeAddressAndSendPayment('base', myAmount, shared_address, peerDeviceAddress, (err, unit) => {
           if (err) return cb(err);
           let arrPayments = [{
              address: shared_address,
              amount: peerAmount,
              asset: 'base'
           }];
           let assocDefinitions = {};
           assocDefinitions[shared_address] = {
              definition: arrDefinition,
              signers: assocSignersByPath
           };
           let objPaymentRequest = {payments: arrPayments, definitions: assocDefinitions};
           let paymentJson = JSON.stringify(objPaymentRequest);
           let paymentJsonBase64 = Buffer.from(paymentJson).toString('base64');
           let paymentRequestCode = 'payment:' + paymentJsonBase64;
           let paymentRequestText = '[...](' + paymentRequestCode + ')';
           cb(null, paymentRequestText, shared_address, timeout);
        });
     }
  });
}
```

Let’s analyze what happens  
**arrSeenConditionPeer** - watching if there was a payment in the contract with the amount same of the user.  
**arrSeenConditionMyInput** - сhecking, did we withdraw bytes? If yes, then allow the user to pick up their bytes.

**arrDefinition** - this is what our contract looks like, let’s look at it in details.  
When we need some address to subscribe we use \[‘address’, address\]  
When we need to check the publication data\_feed we use  
\[‘in data feed’, \[\[conf.oracle\_address\], name, operator, value\]\]

**assocSignersByPath** - in this object, we prescribe the path in the array of the contract before you need to sign and who should sign it. In our example, we sign all ‘address’. The path starts with r-this is the first element in the array, in our case ‘or’ construction. Other examples with signatures will be discussed in the following lessons.

**walletDefinedByAddresses.createNewSharedAddress** - we create a contract address  
**headlessWallet.issueChangeAddressAndSendPayment** - we add the address of the contract and make a payment to it

```javascript
let arrPayments = [{
              address: shared_address,
              amount: peerAmount,
              asset: 'base'
           }];
           let assocDefinitions = {};
           assocDefinitions[shared_address] = {
              definition: arrDefinition,
              signers: assocSignersByPath
           };
           let objPaymentRequest = {payments: arrPayments, definitions: assocDefinitions};
           let paymentJson = JSON.stringify(objPaymentRequest);
           let paymentJsonBase64 = Buffer(paymentJson).toString('base64');
           let paymentRequestCode = 'payment:' + paymentJsonBase64;
           let paymentRequestText = '[your share of payment to the contract](' + paymentRequestCode + ')';
```

This is how we create a message that contains a contract and a payment request.

Create correspondents.js

```javascript
const db = require('ocore/db');

exports.addCorrespondent = (code, name, cb) => {
  let device = require('ocore/device');
 
  function handleCode(code) {
     let matches = code.match(/^([\w\/+]+)@([\w.:\/-]+)#([\w\/+-]+)$/);
     if (!matches)
        return cb("Invalid pairing code");
     let pubkey = matches[1];
     let hub = matches[2];
     let pairing_secret = matches[3];
     if (pubkey.length !== 44)
        return cb("Invalid pubkey length");
    
     acceptInvitation(hub, pubkey, pairing_secret, cb);
  }
 
  function acceptInvitation(hub_host, device_pubkey, pairing_secret, cb) {
     if (device_pubkey === device.getMyDevicePubKey())
        return cb("cannot pair with myself");
     if (!device.isValidPubKey(device_pubkey))
        return cb("invalid peer public key");
     // the correspondent will be initially called 'New', we'll rename it as soon as we receive the reverse pairing secret back
     device.addUnconfirmedCorrespondent(device_pubkey, hub_host, name, (device_address) => {
        device.startWaitingForPairing((reversePairingInfo) => {
           device.sendPairingMessage(hub_host, device_pubkey, pairing_secret, reversePairingInfo.pairing_secret, {
              ifOk: () =>{
                 cb(null, device_address);
              },
              ifError: cb
           });
        });
     });
  }
 
  handleCode(code);
};

exports.findCorrespondentByPairingCode = (code, cb) => {
  let matches = code.match(/^([\w\/+]+)@([\w.:\/-]+)#([\w\/+-]+)$/);
  if (!matches)
     return cb("Invalid pairing code");
  let pubkey = matches[1];
  let hub = matches[2];
 
  db.query("SELECT * FROM correspondent_devices WHERE pubkey = ? AND hub = ?", [pubkey, hub], (rows) => {
     return cb(rows.length ? rows[0] : null);
  });
};

```

We need to tell the oracle to check and publish the weather at the specified time, for this we use this feature.

```javascript
function findOracleAndSendMessage(value, cb) {
  const device = require('ocore/device');
  correspondents.findCorrespondentByPairingCode(conf.oracle_pairing_code, (correspondent) => {
     if (!correspondent) {
        correspondents.addCorrespondent(conf.oracle_pairing_code, 'flight oracle', (err, device_address) => {
           if (err)
              throw new Error(err);
           device.sendMessageToDevice(device_address, 'text', value);
           cb();
        });
     } else {
        device.sendMessageToDevice(correspondent.device_address, 'text', value);
        cb();
     }
  });
}

```

We check whether there is an oracle in our correspondents and if it is not - we add.

That’s all. I also have a homework for you. Complete the oracle and our bot so that oracle informs us that he published and the publication has become stable. After oracle message, notify the user that he lost or won. And if he lost then take the money out of the contract. All code is available on [github](https://github.com/xJeneKx/tutorial-4). Did you like the lesson? Any questions? What topics have caused you difficulties and it is worth talking about them? Write in the comments and I will help you. Also join my telegram chat - [@obytedev](https://t.me/obytedev).

