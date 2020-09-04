---
description: >-
  This guide explains how to request, validate, and use the attested data in
  your chat bot.
---

# Attestation profiles / KYC

Users can have some of their data verified by a trusted third party \(attestor\). The attestor posts an attestation record to the Obyte DAG, this record serves as a proof for relying parties that a user was attested. The attested data itself can be either posted publicly by the attestor, or only a hash of this data is posted while the plain-text data is saved in the wallet of the attested user. In the latter case, the user can disclose the attested data to selected peers, for example to your bot.

This can be used to KYC your users prior to providing a service, see [https://medium.com/byteball/bringing-identity-to-crypto-b35964feee8e](https://medium.com/byteball/bringing-identity-to-crypto-b35964feee8e). You may need it e.g. to comply with regulations or protect against fraud.

ICO bot [https://github.com/byteball/ico-bot](https://github.com/byteball/ico-bot) already uses the attested private profiles in order to allow only verified users to invest and to collect their personal data. You can use its code as reference.

## Requesting private profile

You should point the user to your privacy policy before requesting sensitive personal data.

To request a private profile, you need to send profile-request message to the user. The message includes the list of fields you require the user to disclose. The message will be displayed in the user's chat window as a clickable link, which allows them to select one of the profiles stored in his wallet and send it over to the peer \(your chat bot\).

### Real Name Attestation

This is a profile request message that asks the fields that all [Real Name Attestation](https://github.com/byteball/real-name-attestation) profiles have:

```text
[...](profile-request:first_name,last_name,dob,country,id_type)
```

Here is the full list of available fields:

* `first_name`: first name
* `last_name`: last name
* `dob`: date of birth in YYYY-MM-DD format\_
* `country`: the country that issued the ID \(2-letter ISO code\)
* `us_state`: US state \(only if country=US\)
* `personal_code`: government issued personal code \(not all countries\)
* `id_number`: government issued document ID \(not all documents\)
* `id_type`: ID type
* `id_subtype`: ID sub-type \(not all attestations\)
* `id_expiry`: ID expires at \(not all attestations\)
* `id_issued_at`: ID issued at \(not all attestations\)

### Email Attestation

This is a profile request message that asks the email field that all [Email Attestation](https://github.com/byteball/email-attestation) profiles have:

```text
[...](profile-request:email)
```

### Steem Attestation

This is a profile request message that asks the fields that all [Steem Attestation](https://github.com/byteball/steem-attestation) profiles have:

```text
[...](profile-request:steem_username,reputation)
```

## Receiving private profile

When a user sends you his private profile \(on profile request or by choosing "Insert private profile" from the menu\), you receive it in a chat message that includes:

```text
[...](profile:privateProfileJsonBase64)
```

where `privateProfileJsonBase64` is a base64-encoded JSON of the private profile object. You can easily find the profile in an incoming message using regular expression:

```javascript
let arrProfileMatches = text.match(/\(profile:(.+?)\)/);
```

Then, decode the profile using `private_profile.js` module in `ocore`:

```javascript
const privateProfile = require('ocore/private_profile.js');
let privateProfileJsonBase64 = arrProfileMatches[1];
let objPrivateProfile = privateProfile.getPrivateProfileFromJsonBase64(privateProfileJsonBase64);
```

`objPrivateProfile` object has 3 fields:

```javascript
objPrivateProfile {
    unit: "...", // attestation unit
    payload_hash: "...", // pointer to attestation payload in this unit
    src_profile: object // attested fields
}
```

Next, you need to validate this object and extract information about the attested and attestor addresses:

```javascript
privateProfile.parseAndValidatePrivateProfile(objPrivateProfile, (err, address, attestor) => {
    if (err) {
        device.sendMessageToDevice(from_address, 'text', err);
    }
    // validated
    // ...
});
```

This function verifies that the provided profile matches the hash stored on the DAG by the attestor. It also returns the user's address `address` \(hence you don't need to ask the user about his address, the profile is enough\) and the attestor address `attestor` \(make sure it is on the list of attestors you trust for each type of attestations\).

The `src_profile` field of `objPrivateProfile` contains an associative array of attested fields. But not all fields have to be disclosed by the user.

* If a field is disclosed, the value of `objPrivateProfile[field]` is an array of two elements: the plain-text value of the field and blinding. Blinding is a random string generated by the attestor when creating the profile, it serves to protect the private data from brute force attacks \(the profile data is too predictable and can be easily checked against known hashes\).
* If the field is not disclosed, the value of `objPrivateProfile[field]` is a hash of plain-text value and blinding.

To extract just the disclosed data and remove all the bindings, use

```javascript
let assocPrivateData = privateProfile.parseSrcProfile(objPrivateProfile.src_profile);
```

You should double check if the user sent all the required fields because users can send private profiles on profile request \(required fields in locked state\) or by choosing "Insert private profile" from the menu \(none of the fields in locked state\).

To save the received private profile of previously mentioned Real Name Attestation profile request, use code like this:

```javascript
let required_fields = ['first_name', 'last_name', 'dob', 'country', 'id_type'];
if (required_fields.every(key => key in assocPrivateData)) {
    privateProfile.savePrivateProfile(objPrivateProfile, address, attestor);
}
else {
    device.sendMessageToDevice(from_address, 'text',
            'Please share all the required fields.');
}
```

It will save the profile in the tables `private_profiles` and `private_profile_fields`, which you can later query to read the data.

## Sending private profile

If your bot is setting up [prosaic contracts](contracts/prosaic-contracts.md) between 2 users then they need to get each others private profiles. All you need to do is first request the profiles from both users, then receive and save the profiles from both users and then send the first user and second user's profile and the second user a first user's profile.

```javascript
db.query("SELECT unit, payload_hash, src_profile \
		FROM private_profiles \
		WHERE address = ? AND attestor_address = ? \
		LIMIT 1;", [wallet_address_1, attestor], (rows) => {
	if (rows.length === 0) {
		return device.sendMessageToDevice(device_address_1, 'text',
				'Please provide your private email \
				[...](profile-request:first_name,last_name)');
	}
	let objPrivateProfile = {
		unit: rows[0].unit,
		payload_hash: rows[0].payload_hash,
		src_profile: JSON.parse(rows[0].src_profile)
	};
	let privateProfileJsonBase64 = Buffer.from(JSON.stringify(objPrivateProfile)).toString('base64');
	device.sendMessageToDevice(device_address_2, 'text',
			'Save the private profile of the other side of the contract. \
			[...](profile:'+ privateProfileJsonBase64 +')');
});
```

## Retrieving public attestation

[Real Name Attestation](https://github.com/byteball/real-name-attestation) profiles are all private, but [Email Attestation](https://github.com/byteball/email-attestation) and [Steem Attestation](https://github.com/byteball/steem-attestation) bots let user to choose whether they like to publish the data publicly or privately and [Username Attestation](https://github.com/byteball/username-attestation) bot publishes attestations only publicly. So, if you are asking attested email address from the user, you should let them do it with public attestation too.

```javascript
const device = require('ocore/device.js');
let challenge = 'This is my address.';
device.sendMessageToDevice(from_address, 'text', 'Please provide your private email [...](profile-request:email) \
        or insert the wallet address, which has publicly attested email address [...](sign-message-request:'+ challenge +'). \
        \nIf you haven\'t verified your email address yet then please do it with Email Attestation bot from Bot Store.');
```

For full nodes that is easy, you ask user to [sign some message](verifying-address-with-signed-message.md), after which you get a wallet address that they own. Then you just query your full node database for that data:

```javascript
const email_attestor = 'H5EZTQE7ABFH27AUDTQFMZIALANK6RBG';
/*
convert base64 signed message into JSON object
like it is described in Wallet address verification documentation.
*/

validation.validateSignedMessage(objSignedMessage, err => {
    if (err)
        return device.sendMessageToDevice(from_address, 'text', err);
    if (objSignedMessage.signed_message !== challenge)
        return device.sendMessageToDevice(from_address, 'text',
                "You signed a wrong message: "+objSignedMessage.signed_message+", expected: "+challenge);

    db.query("SELECT unit, field, value FROM attested_fields \
            WHERE attestor_address = ? AND address = ?;",
            [email_attestor, objSignedMessage.authors[0].address], (rows) => {
        if (rows.length === 0) return;
        rows.forEach((row) => {
            if (row.field == 'email') {
                device.sendMessageToDevice(from_address, 'text',
                        'Found: '+ row.value);
            }
        });
    });
});
```

On the light node, it is also possible, but more complicated. You need to ask the Hub for all the attestation for user address and then you would need to request proofs for those attestation units:

```javascript
const email_attestor = 'H5EZTQE7ABFH27AUDTQFMZIALANK6RBG';
/*
convert base64 signed message into JSON object
like it is described in Wallet address verification documentation.
*/

validation.validateSignedMessage(objSignedMessage, err => {
    if (err)
        return device.sendMessageToDevice(from_address, 'text', err);
    if (objSignedMessage.signed_message !== challenge)
        return device.sendMessageToDevice(from_address, 'text',
                "You signed a wrong message: "+objSignedMessage.signed_message+", expected: "+challenge);

    const network = require('ocore/network.js');
    network.requestFromLightVendor('light/get_attestations', {
        address: objSignedMessage.authors[0].address
    },
    (ws, request, response) => {
        if (response.error){
            console.error('light/get_attestations failed: '+response.error);
            return;
        }
        let arrAttestations = response;
        if (!Array.isArray(arrAttestations)){
            console.error('light/get_attestations response is not an array: '+response);
            return;
        }
        if (arrAttestations.length === 0){
            return device.sendMessageToDevice(from_address, 'text',
                    "Not an attested address.");
        }
        console.log('attestations', arrAttestations);
        let arrUnits = arrAttestations.map(attestation => attestation.unit);
        network.requestProofsOfJointsIfNewOrUnstable(arrUnits, err => {
            if (err){
                console.log('requestProofsOfJointsIfNewOrUnstable failed: '+err);
                return;
            }
            arrAttestations.forEach( (attestation) => {
                if (attestation.attestor_address === email_attestor) {
                    device.sendMessageToDevice(from_address, 'text',
                            'Found: '+ attestation.profile.email);
                }
            });
        });
    });
});
```

## Creating attestation profiles

Above examples here show how to use different existing profiles from other attestation bots, but if you would like to create your own attestation bot, it is recommended to click on the links of these bots on this article and take a look how they have been done from the source code. There is also article ["How to create private/public attestation profile" on Sending data to DAG ](payments/data.md#attestation-profile)page.

