---
description: >-
  You can store arbitrary data (any sequence of bytes) in Obyte public DAG. Some
  special types of data also supported, like 'text', 'profile', 'poll', etc.
---

# Sending data to DAG

## Data with any structure

To send `data` to DAG database, code below is a minimal that you can use. Since the `opts.messages` is array, it means that all the examples on this page can be changed to multi-message transactions.

```javascript
const headlessWallet = require('headless-obyte');
const eventBus       = require('ocore/event_bus.js');

function postData() {
    let json_data = {
        age: 78.90,
        props: {
            sets: [
                '0bbb',
                'zzz',
                1/3
            ]
        }
    };
    let objMessage = {
        app: 'data',
        payload_location: 'inline',
        payload: json_data
    };
    let opts = {
        messages: [objMessage]
    };

    ​headlessWallet.issueChangeAddressAndSendMultiPayment(opts, (err, unit) => {
        if (err){
            /*
            something went wrong,
            maybe put this transaction on a retry queue
            */
            return;
        }
        // handle successful payment
    });
}

eventBus.on('headless_wallet_ready', postData);
```

AAs can be [triggered with data](../autonomous-agents/getting-started-guide.md#passing-data-to-aa) by manually entering the key/value pairs to GUI wallet, [prompting users to submit pre-filled Send screen](../byteball-protocol-uri.md#requesting-to-send-data-to-aa), making the [AA to trigger another AA](../autonomous-agents/getting-started-guide.md#sending-data) or with the [script from headless wallet](https://github.com/byteball/headless-obyte/blob/master/tools/send_data_to_aa.js).

## Key-value data feed

Previous example shows how to send any data to DAG, but if you would want to store data that is searchable and can be used in [smart-contracts](../contracts/smart-contracts.md) or [autonomous agents](../autonomous-agents/getting-started-guide.md) then you would need to post it as key-value pairs and with app type `data_feed`. If we also send it always from the same first address in our wallet then we have basically become an oracle.

```javascript
const headlessWallet = require('headless-obyte');
const eventBus = require('ocore/event_bus.js');

function postDataFeed(first_address) {
    let json_data = {
        time: new Date().toString(), 
        timestamp: Date.now()
    };
    let objMessage = {
        app: 'data_feed',
        payload_location: 'inline',
        payload: json_data
    };
    let opts = {
        paying_addresses: [first_address],
        messages: [objMessage]
    };

    ​headlessWallet.issueChangeAddressAndSendMultiPayment(opts, (err, unit) => {
        if (err){
            /*
            something went wrong,
            maybe put this transaction on a retry queue
            */
            return;
        }
        // handle successful payment
    });
}

eventBus.once('headless_wallet_ready', () => {
    headlessWallet.readFirstAddress(postDataFeed);
});
```

## Profile about yourself

Anybody on Obyte network can post some data about themselves, which will be linked to their address. This app type is called `profile`. Payload for that message can contain any keys. For example, [obyte.io](https://obyte.io/edit-profile) web service lets you post a profile with keys like this:

```javascript
let json_data = {
    name:     'a', 
    about:    'b',
    location: 'c',
    website:  'd',
    email:    'e'        
};
let objMessage = {
    app: 'profile',
    payload_location: 'inline',
    payload: json_data
};
```

## Attestation profile

Attestation profile is similar to previous one with one difference that `attestation` type message are posted by somebody and they describe somebody else. These profiles are used to do KYC by [asking user to provide data about themselves from trusted attestor](../private-profiles.md#requesting-private-profile).

### Private/Public attestations

[Real Name Attestation](https://github.com/byteball/real-name-attestation) bot creates an `attestation` profile, which keeps all the data private on user device while makes it possible for the others to recognize unique users by `user_id` \(it's a salted hash of `first_name`, `last_name`, `dob` and `country`\) and also makes it possible to validate with `profile_hash` if user provides them a private profile.

```javascript
const conf = require('ocore/conf.js');
const objectHash = require('ocore/object_hash.js');

let bPublic = false; // ask user if they want public or private profile

function hideProfile(profile, bPublic = false){
    const composer = require('ocore/composer.js');
    let hidden_profile = {};
    let src_profile = {};
    for (let field in profile){
        let value = profile[field];
        let blinding = composer.generateBlinding();
        let hidden_value = objectHash.getBase64Hash([value, blinding], true);
        hidden_profile[field] = hidden_value;
        src_profile[field] = [value, blinding];
    }
    let shortProfile = {
        first_name: profile.first_name,
        last_name:  profile.last_name,
        dob:        profile.dob,
        country:    profile.country,
    };
    let public_profile;
    if (bPublic) {
        public_profile = profile;
        public_profile.user_id = objectHash.getBase64Hash([shortProfile, conf.salt]);
        return [public_profile, null];
    }
    else {
        public_profile = {
            profile_hash: objectHash.getBase64Hash(hidden_profile, true),
            user_id: objectHash.getBase64Hash([shortProfile, conf.salt])
        };
        return [public_profile, src_profile];
    }
}

let profile = {
    first_name: "", // required
    last_name:  "", // required
    dob:        "", // required
    country:    "", // required
    us_state:   "", // optional
    id_type:    ""  // optional
};
for (let field in profile){
    if (typeof profile[field] === 'undefined' || profile[field] === null || profile[field] === '') {
        // remove negative values, except number 0
        delete profile[field];
    }
    else {
        // convert everything else to string
        profile[field] = String(profile[field]);
    }
}
let [public_data, src_profile] = hideProfile(profile, bPublic);
let json_data = {
    address: 'ABCDEF...',
    profile: public_data
};
let objMessage = {
    app: 'attestation',
    payload_location: 'inline',
    payload: json_data
};
```

In case user would want to make their profile public like with [Email Attestation](https://github.com/byteball/email-attestation) and [Steem Attestation](https://github.com/byteball/steem-attestation), `true` could be sent as a second parameter of `hideProfile()` function. To avoid people doxxing themselves, the possibility to make public attestation with Real Name Attestation has not been included. This choice is only on Email Attestation and Steem Attestation.

In order for the user to share private profile with others, your bot should also [send the private profile to user's wallet](../private-profiles.md#sending-private-profile).

### Public only attestations

If there is no plan to provide private attestations and all the attestation will always be public then there is no need for `user_id`. Then the attestation profile can be as simple as this, but the attestator needs to make sure that the same username attestation is not done to multiple users. One example of that kind is [Username Attestation bot](https://github.com/byteball/username-attestation).

```javascript
let json_data = {
    address: 'ABCDEF...',
    profile: {
        username: ""
    }    
};
let objMessage = {
    app: 'attestation',
    payload_location: 'inline',
    payload: json_data
};
```

For public attestation, there is not private profile to send to user's wallet because the bots who need information about the user could simply just [retrieve the attestation profile from the DAG](../private-profiles.md#retrieving-public-attestation).

## Poll question and choices

Anybody can post a poll and let others vote on it, this can be done like this data and it will also appear on [obyte.io](https://obyte.io/polls) web service from where it will direct users to [Poll chat bot](https://github.com/byteball/poll-bot).

```javascript
let json_data = {
    question: 'abc?', 
    choices: [
        'a',
        'b',
        'c',
    ]        
};
let objMessage = {
    app: 'poll',
    payload_location: 'inline',
    payload: json_data
};
```

## Plain text data

And last and not the least, it is possible to post plain text to DAG too.

```javascript
let text_data = '';
let objMessage = {
    app: 'text',
    payload_location: 'inline',
    payload: text_data
};
```

## More examples

Shorter code examples using composer library are available in ["/tools" folder of headless-wallet](https://github.com/byteball/headless-obyte/tree/master/tools).

If you wish to send data to DAG database periodically then full code examples for [price oracle](https://github.com/byteball/byteball-data-feed), [bitcoin oracle](https://github.com/byteball/btc-oracle) or [sports oracle](https://github.com/byteball/sports-oracle) can be found on Github.

