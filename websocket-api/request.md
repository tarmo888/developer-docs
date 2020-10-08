---
description: 'This is a message type, which gets a reply with a response type message.'
---

# Request

Example:

```javascript
const network = require('ocore/network.js');
const ws = network.getInboundDeviceWebSocket(device_address);

// function parameters: websocket, command, params, bReroutable, callback
var tag = network.sendRequest(ws, 'get_witnesses', null, false, getParentsAndLastBallAndWitnessListUnit);

function getParentsAndLastBallAndWitnessListUnit(ws, req, witnesses) {
  var params = {
    witnesses: witnesses
  };
  network.sendRequest(ws, 'light/get_parents_and_last_ball_and_witness_list_unit', params, false,
    function(ws, req, response) {
      console.log(response);
    }
  );
}

//you can have your own timeout logic and delete a pending request like this
setTimeout(function() {
  var deleted = network.deletePendingRequest(ws, tag); // true if request was deleted
}, 30*1000);
```

Following is a list of `request` type JSON messages that are sent over the network:

### **Get node list**

This requests returns response with nodes that accept incoming connections.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'get_peers'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [
          "wss://relay.bytes.cash/bb",
          "wss://hub.obytechina.org/bb",
          "wss://hub.obyte.connectory.io/bb",
          "wss://odex.ooo/bb",
          ...
        ]
    }
}
```
{% endtab %}
{% endtabs %}

### **Get a list of witnesses**

This requests returns response with list of witnesses.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'get_witnesses'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [
          "4GDZSXHEFVFMHCUCSHZVXBVF5T2LJHMU",
          "BVVJ2K7ENPZZ3VYZFWQWK7ISPCATFIW3",
          "DJMMI5JYA5BWQYSXDPRZJVLW3UGL3GJS",
          ...
        ]
    }
}
```
{% endtab %}
{% endtabs %}

### **Get transaction data**

This requests returns response with unit data. Example shows a unit, which created a new asset.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'get_joint',
        params: '0xXOuaP5e3z38TF5ooNtDhmwNkh1i21rBWDvrrxKt0U='
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
          "joint": {
            "unit": {
              "unit": "0xXOuaP5e3z38TF5ooNtDhmwNkh1i21rBWDvrrxKt0U=",
              "version": "1.0",
              "alt": "1",
              "witness_list_unit": "oj8yEksX9Ubq7lLc+p6F2uyHUuynugeVq4+ikT67X6E=",
              "last_ball_unit": "AV4qvQ5JuI7J3EO5pFD6UmMu+j0crPbH17aJfigozrc=",
              "last_ball": "Ogq4+uR6LaDspK89MmyydxRchgdN/BYwXvWXwNCMubQ=",
              "headers_commission": 344,
              "payload_commission": 607,
              "main_chain_index": 2336026,
              "timestamp": 1524844417,
              "parent_units": [
                "IFiHjYbzuAUC9TwcTrmpvezjTKBl6nIXjgjoKuVyCz0="
              ],
              "authors": [
                {
                  "address": "AM6GTUKENBYA54FYDAKX2VLENFZIMXWG",
                  "authentifiers": {
                    "r": "MBWlL31OkibUXhoxmwIcWB/fAx1uWdNTE8PYTBNeN3w+tev4N1anOXjGtXiGW4whW3PfJeTO9fA5WxwyqK/m2w=="
                  }
                }
              ],
              "messages": [
                {
                  "app": "data",
                  "payload_hash": "MiuMtTSgP0brPQijsRc0igb+dxcrkLjXWRE253AE/S8=",
                  "payload_location": "inline",
                  "payload": {
                    "asset": "IYzTSjJg4I3hvUaRXrihRm9+mSEShenPK8l8uKUOD3o=",
                    "decimals": 0,
                    "name": "WCG Point by Byteball",
                    "shortName": "WCG Point",
                    "issuer": "Byteball",
                    "ticker": "WCG",
                    "description": "WCG Point is a honorific token, a recognition of contributing to World Community Grid projects. The token is not transferable, therefore, it cannot be sold and the balance reflects a lifetime contribution to WCG. Some services might choose to offer a privilege to users with large balance of this token."
                  }
                },
                {
                  "app": "payment",
                  "payload_hash": "FaVxPdeRKTgcO/LCDtu/YUZego8xyrnpUMF3V/sujr8=",
                  "payload_location": "inline",
                  "payload": {
                    "inputs": [
                      {
                        "unit": "1p85M8VSRDaAkNoCAdtMrv6cVGpPNNeVJpLzOkzWLk8=",
                        "message_index": 1,
                        "output_index": 0
                      }
                    ],
                    "outputs": [
                      {
                        "address": "AM6GTUKENBYA54FYDAKX2VLENFZIMXWG",
                        "amount": 17890
                      }
                    ]
                  }
                }
              ]
            },
            "ball": "Ef36OjK5m6lQMII4S9MN3iGtQcsvTzzQFLKQZ44UBCg="
          }
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Send transaction data**

This requests returns response whether composed unit was accepted or not. Unit object can be composed with `ocore/composer.js`.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'post_joint',
        params: {Object}
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'accepted'
    }
}
```
{% endtab %}
{% endtabs %}

### **Send heartbeat that you don't sleep**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'heartbeat'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: null
    }
}
```

```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'sleep'
    }
}
```
{% endtab %}
{% endtabs %}

### **Subscribe to transaction data**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: "request",
    content: {
        tag: tag,
        command: "subscribe",
        params: {
            subscription_id: crypto.randomBytes(30).toString('base64'),
            last_mci: 0,
            library_version: "0.1"
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: "subscribed"
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light, cannot subscribe you to updates",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Synchronous transaction data**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'catchup',
        params: {
            witnesses: witnesses,
            last_stable_mci: last_stable_mci,
            last_known_mci: last_known_mci
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
            status: 'current'
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Get the hash tree**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'get_hash_tree',
        params: {
            from_ball: from_ball,
            to_ball: to_bal      
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
          "balls": [
            {
              "unit": "c2teVop7xa1BmH1LPvytvOKU8HHTrprGWQw+uHlWPHo=",
              "ball": "ht1QP48paWg5hpjx+Nbd/DSRFT8WlbQKk9+Uum0/tso=",
              "parent_balls": [
                "aEU1WiY9FQ9ihv9cKkX/EHWDxYYVaWYs2AL1yxyYZAQ="
              ]
            },
            {
              "unit": "tPbC6QLeweGuiGYRPsIhfd0TQWXWASYBJJUPGa9AOfw=",
              "ball": "ba5wFB+gewdRX6nSpxt6+Nt8PaRen4pLIYg3jC6EvIw=",
              "parent_balls": [
                "ht1QP48paWg5hpjx+Nbd/DSRFT8WlbQKk9+Uum0/tso="
              ]
            },
            {
              "unit": "J7quEBEZ5ksq+0vK4cku7NtWkZ4Kxb8aHCu66RkN2eU=",
              "ball": "4B+R+gW91BPmTqxF3G10oWmF4tg5LiHdpMQcL1ezqCo=",
              "parent_balls": [
                "ba5wFB+gewdRX6nSpxt6+Nt8PaRen4pLIYg3jC6EvIw="
              ]
            },
            ...
          ]
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Get the main chain index**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'get_last_mci'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 3795741
    }
}
```
{% endtab %}
{% endtabs %}

### Send message to client that is connected to hub

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/deliver',
        params: {
            to: device_address,
            pubkey: pubkey,
            signature: signature,
            encrypted_package: encrypted_message
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'accepted'
    }
}
```
{% endtab %}
{% endtabs %}

### **Get temporary public key**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/get_temp_pubkey',
        params: permanent_pubkey
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'TEMP_PUBKEY_PACKAGE'
    }
}
```
{% endtab %}
{% endtabs %}

### **Update temporary public key**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/temp_pubkey',
        params: {
            temp_pubkey: temp_pubkey,
            pubkey: permanent_pubkey,
            signature: signature
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'updated'
    }
}
```
{% endtab %}
{% endtabs %}

### **Enable notifications**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/enable_notification',
        params: registrationId
    }
}
```
{% endtab %}

{% tab title="request.json" %}
```
{
    type: 'response',
    content: {
        tag: tag,
        response: 'ok'
    }
}
```
{% endtab %}
{% endtabs %}

### **Disable notifications**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/disable_notification',
        params: registrationId
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'ok'
    }
}
```
{% endtab %}
{% endtabs %}

### **Get list of chat bots**

This requests returns response with chat bots of connected hub.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/get_bots'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
	type: 'response',
	content: {
		tag: tag,
		response: [
			{
				"id": 29,
				"name": "Buy Bytes with Visa or Mastercard",
				"pairing_code": "A1i/ij0Na4ibEoSyEnTLBUidixtpCUtXKjgn0lFDRQwK@byteball.org/bb#0000",
				"description": "This bot helps to buy Bytes with Visa or Mastercard.  The payments are processed by Indacoin.  Part of the fees paid is offset by the reward you receive from the undistributed funds."
			},
			{
				"id": 31,
				"name": "World Community Grid linking bot",
				"pairing_code": "A/JWTKvgJQ/gq9Ra+TCGbvff23zqJ9Ec3Bp0XHxyZOaJ@byteball.org/bb#0000",
				"description": "Donate your device’s spare computing power to help scientists solve the world’s biggest problems in health and sustainability, and earn some Bytes in the meantime.  This bot allows you to link your Byteball address and WCG account in order to receive daily rewards for your contribution to WCG computations.\n\nWCG is an IBM sponsored project, more info at https://www.worldcommunitygrid.org"
			},
			{
			"id": 36,
				"name": "Username registration bot",
				"pairing_code": "A52nAAlO05BLIfuoZk6ZrW5GjJYvB6XHlCxZBJjpax3c@byteball.org/bb#0000",
				"description": "Buy a username and receive money to your @username instead of a less user-friendly cryptocurrency address.\n\nProceeds from the sale of usernames go to Byteball community fund and help fund the development and promotion of the platform."
			},
			...
		]
	}
}
```
{% endtab %}
{% endtabs %}

### **Get asset metadata**

This requests returns response with asset metadata \(unit and registry\). Example gets WCG Point asset metadata.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'hub/get_asset_metadata',
        params: 'IYzTSjJg4I3hvUaRXrihRm9+mSEShenPK8l8uKUOD3o='
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
            "metadata_unit": "0xXOuaP5e3z38TF5ooNtDhmwNkh1i21rBWDvrrxKt0U=",
            "registry_address": "AM6GTUKENBYA54FYDAKX2VLENFZIMXWG",
            "suffix": null
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Get transaction history**

This requests returns response with transaction history of specified addresses.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_history',
        params: {
            witnesses: witnesses,
            requested_joints: joints,
            addresses: addresses
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {Object}
    }
}
```
{% endtab %}
{% endtabs %}

### **Get chain link proofs**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_link_proofs',
        params: units
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [Array]
    }
}
```
{% endtab %}
{% endtabs %}

### **Get the parent unit and the witness unit**

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_parents_and_last_ball_and_witness_list_unit',
        params: {
            witnesses: witnesses
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
          "parent_units": [
            "QEdpDamxpez8dlOE4nF8TWwHs+efokBXlxQHK27/y4g="
          ],
          "last_stable_mc_ball": "xF0K/NKO6CGMU6XeBGXzurEcCajomfZMOAU4XmAy+6o=",
          "last_stable_mc_ball_unit": "R/RosYwuXJ/mXw5TTfWLVLHdtnnzaKn5EgJkDfzagcs=",
          "last_stable_mc_ball_mci": 3795817,
          "witness_list_unit": "J8QFgTLI+3EkuAxX+eL6a0q114PJ4h4EOAiHAzxUp24="
        }
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### **Get specific attestation unit**

This requests returns response with attestation units for specified field and value.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_attestation',
        params: {
            attestor_address: 'FZP4ZJBMS57LYL76S3J75OJYXGTFAIBL',
            field: 'name',
            value: 'tarmo888'
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 'jeiBABcZI5fjyIPHkpb2PipLHzjUgafoPd0b6bdsGUI='
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Get all attestation data about address

This requests returns response with all the attestation data about specific address.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_attestations',
        params: {
            address: 'MNWLVYTQL5OF25DRRCR5DFNYXLSFY43K'
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [
          {
            "unit": "jeiBABcZI5fjyIPHkpb2PipLHzjUgafoPd0b6bdsGUI=",
            "attestor_address": "FZP4ZJBMS57LYL76S3J75OJYXGTFAIBL",
            "profile": {
              "name": "tarmo888"
            }
          }
        ]
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Pick divisible inputs for amount

This requests returns response with spendable inputs for specified asset, amount and addresses.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/pick_divisible_coins_for_amount',
        params: {
            asset: '',
            addresses: addresses,
            last_ball_mci: last_ball_mci,
            amount: amount,
            spend_unconfirmed: 'own'
        }
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {Object}
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Get address definition

This requests returns response with address definition \(smart-contracts have address smart-contract definition only after somebody has spent from it\).

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_definition',
        params: 'JEDZYC2HMGDBIDQKG3XSTXUSHMCBK725'
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [
          "sig",
          {
            "pubkey": "Aiy5z+jM1ySTZl1Qz1YZJouF7tU6BU++SYc/xe0Rj5OZ"
          }
        ]
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Get balances of addresses

This request returns a response with balances of one or more addresses.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_balances',
        params: [
            'JEDZYC2HMGDBIDQKG3XSTXUSHMCBK725',
            'UENJPVZ7HVHM6QGVGT6MWOJGGRTUTJXQ'
        ]
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: {
          "JEDZYC2HMGDBIDQKG3XSTXUSHMCBK725": {
            "base": {
              "stable": 1454617324,
              "pending": 2417784
            }
          },
          "UENJPVZ7HVHM6QGVGT6MWOJGGRTUTJXQ": {
            "base": {
              "stable": 0,
              "pending": 6
            }
          }
        }
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Get profile unit IDs of addresses

It is possible for users to post a profile information about themselves, this request returns a response with all the profile data units.

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'light/get_profile_units',
        params: [
            'MNWLVYTQL5OF25DRRCR5DFNYXLSFY43K'
        ]
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: [
          "gUrJfmdDeYhTKAHM5ywydHXpvcensbkv8TPQLuPG3b0="
        ]
    }
}
```

```javascript
{
    type: "response",
    content: {
        tag: tag,
        response: {
            error: "I'm light myself, can't serve you",
        }
    }
}
```
{% endtab %}
{% endtabs %}

### Custom Request

You can add your own communication protocol on top of the Obyte one. See event [there](../list-of-events.md#event-for-custom-request).

{% tabs %}
{% tab title="request.json" %}
```javascript
{
    type: 'request',
    content: {
        tag: tag,
        command: 'custom',
        params: params
    }
}
```
{% endtab %}

{% tab title="response.json" %}
```javascript
{
    type: 'response',
    content: {
        tag: tag,
        response: 0 || 'some response' || {Object} || [Array]
    }
}
```
{% endtab %}
{% endtabs %}

