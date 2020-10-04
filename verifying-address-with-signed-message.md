---
description: >-
  If you need to verify that a user owns a particular address, you can do it by
  offering to sign a message.
---

# Address signing/verification

You can also verify ownership of address by requesting a payment from their address and this method is more appropriate if you need to receive a payment anyway.

To request the user to sign a message, send this message in chat:

```text
[...](sign-message-request:challenge)
```

where `challenge` is the message to be signed. This request will be displayed in the user's wallet as a link that the user can click and confirm signing. Once the user signs it, your chat bot receives a specially formatted message:

```text
[...](signed-message:base64_encoded_text)
```

You can parse and validate it, here is an example how to verify user's account address:

```javascript
// verify posted addresss
const validationUtils = require('ocore/validation_utils');
const device = require('ocore/device.js');
var sessionData = [];

eventBus.on('text', (from_address, text) => {
	let arrSignedMessageMatches = text.match(/\(signed-message:(.+?)\)/);
	
	if (validationUtils.isValidAddress(text)) {
		sessionData[from_address] = text;
		let challenge = 'My address is '+text;
		return device.sendMessageToDevice(from_address, 'text',
				'[...](sign-message-request:'+ challenge  +')');
	}
	else if (arrSignedMessageMatches){
		let signedMessageBase64 = arrSignedMessageMatches[1];
		let validation = require('ocore/validation.js');
		let signedMessageJson = Buffer.from(signedMessageBase64, 'base64').toString('utf8');
		try{
			var objSignedMessage = JSON.parse(signedMessageJson);
		}
		catch(e){
			return null;
		}
		validation.validateSignedMessage(objSignedMessage, err => {
			let user_address = sessionData[from_address];
			let challenge = 'My address is '+user_address;
			if (err)
				return device.sendMessageToDevice(from_address, 'text', err);
			if (objSignedMessage.signed_message !== challenge)
				return device.sendMessageToDevice(from_address, 'text',
					"You signed a wrong message: "+objSignedMessage.signed_message+", expected: "+challenge);
			if (objSignedMessage.authors[0].address !== user_address)
				return device.sendMessageToDevice(from_address, 'text',
					"You signed the message with a wrong address: "+objSignedMessage.authors[0].address+", expected: "+user_address);
			
			// all is good, address proven, continue processing
		});
	}
});
```

The above code also checks that the user signed the correct message and with the correct address.

`objSignedMessage` received from the peer has the following fields:

* `signed_message`: \(string\) the signed message
* `authors`: array of objects, each object has the following structure \(similar to the structure of `authors` in a unit\):
  * `address`: \(string\) the signing address
  * `definition`: \(array\) definition of the address
  * `authentifiers`: \(object\) signatures for different signing paths

To validate the message, call `validation.validateSignedMessage` as in the example above.

Note that the challenge that you offer to sign must be both clear to the user and sufficiently unique. The latter is required to prevent reuse of a previously saved signed message.

