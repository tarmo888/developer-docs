---
description: >-
  On Obyte, users can issue new assets and define rules that govern their
  transferability.
---

# Issuing assets on Obyte

To start a user-defined asset on Obyte, you need to complete two steps

* Define your asset. Asset definition is a special type of message \(distinct from payments\) that you send to the Obyte DAG
* Issue the asset by sending the 1st transaction that spends it

## Asset ID

After an asset is defined, the hash of the unit where it was defined becomes the unique ID of the asset. The asset is referenced by this ID in future payments and definitions \(smart contracts\). There are no user-assignable names of assets in the protocol, but there are independent registries that link asset IDs to user-friendly names for a fee. One such registry is [Obyte Asset Registry](https://asset.obyte.app/), another decentralized registry is [tokens.ooo](https://tokens.ooo/)

If you are not a developer, but wish to issue an asset on Obyte then you can also use [Obyte Asset Registry](https://asset.obyte.app/) to do that without writing any code. If Obyte Asset Registry is too limited for your needs or you need to issue an asset on TESTNET then continue reading.

## Defining a new asset

See a sample of asset-defining transaction at [https://github.com/byteball/headless-obyte/blob/master/tools/create\_asset.js](https://github.com/byteball/headless-obyte/blob/master/tools/create_asset.js).

To define a new asset you need a headless wallet. In your dependencies:

```javascript
"dependencies": {
	"headless-obyte": "git+https://github.com/byteball/headless-obyte.git",
	"ocore": "git+https://github.com/byteball/ocore.git"
	// .....
}
```

Call this function to create a new asset definition and broadcast it:

```javascript
var headlessWallet = require('headless-obyte');
var network = require('ocore/network.js');
var composer = require('ocore/composer.js');

var my_address = ''; // set definer address.
var asset = {}; // defined asset here.
composer.composeAssetDefinitionJoint(my_address, asset, headlessWallet.signer,
    {
        ifError: console.error,
        ifNotEnoughFunds: console.error,
        ifOk: function(objJoint, assocPrivatePayloads, composer_unlock) {
            network.broadcastJoint(objJoint);
            console.error('==== Asset ID:'+ objJoint.unit.unit);
        }
    }
);
```

Here `my_address` is your address you use to post the asset-defining message. The address must be funded to pay at least the transaction fees \(about 1000 bytes\).

`asset` is a javascript object that describes the properties of your asset. Here is an example:

```javascript
var asset = {
	cap: 1000000,
	is_private: false,
	is_transferrable: true,
	auto_destroy: false,
	fixed_denominations: false,
	issued_by_definer_only: true,
	cosigned_by_definer: false,
	spender_attested: false
}
```

`cap` is the total number of coins that can be issued \(money supply\). If omitted, the number is unlimited.

`is_private`: indicates whether the asset is private \(such as blackbytes\) or publicly traceable \(similar to bytes\).

`is_transferrable`: indicates whether the asset can be freely transferred among arbitrary parties or all transfers should involve the definer address as either sender or recipient. The latter can be useful e.g. for loyalty points that cannot be resold.

`auto_destroy`: indicates whether the asset is destroyed when it is sent to the definer address.

`fixed_denominations`: indicates whether the asset exists as coins \(banknotes\) of a limited set of denominations, similar to blackbytes. If it is `true`, the definition must also include property `denominations`, which is an array of all denominations and the number of coins of that denomination:

```javascript
var asset = {
	// ....
	fixed_denominations: true,
	denominations: [
		{denomination: 1, count_coins: 1e10},
		{denomination: 2, count_coins: 2e10},
		{denomination: 5, count_coins: 1e10},
		{denomination: 10, count_coins: 1e10},
		{denomination: 20, count_coins: 2e10},
		{denomination: 50, count_coins: 1e10},
		{denomination: 100, count_coins: 1e10},
		{denomination: 200, count_coins: 2e10},
		{denomination: 500, count_coins: 1e10},
		{denomination: 1000, count_coins: 1e10},
		{denomination: 2000, count_coins: 2e10},
		{denomination: 5000, count_coins: 1e10},
		{denomination: 10000, count_coins: 1e10},
		{denomination: 20000, count_coins: 2e10},
		{denomination: 50000, count_coins: 1e10},
		{denomination: 100000, count_coins: 1e10}
	]
};
```

`issued_by_definer_only`: indicates whether the asset can be issued only by the definer address. If `false`, anyone can issue the asset, in this case `cap` must be unlimited.

`cosigned_by_definer`: indicates whether each operation with the asset must be cosigned by the definer address. Useful for regulated assets where the issuer \(bank\) wants to perform various compliance checks \(such as the funds are not arrested by a court order\) prior to approving a transaction. This could also be used to allow fee-less asset transfers for users \(paid by definer\), but wallets don't support that yet. 

`spender_attested`: indicates whether the spender of the asset must be attested by one of approved attestors. Also useful for regulated assets e.g. to limit the access to the asset only to KYC'ed users. If `true`, the definition must also include the list of approved attestor addresses:

```javascript
var asset = {
	// ....
	spender_attested: true,
	attestors: ["ADDRESS1", "ADDRESS2"]
};
```

The definition can also include two optional properties `issue_condition` and `transfer_condition` which specify the restrictions when the asset can be issued and transferred. They evaluate to a boolean and are coded in the same [smart contract language](contracts/reference.md) as address definitions.

```javascript
var asset = {
	// ....
	issue_condition: ["and", [
		["has one", {what: "output", asset: "this asset"}],
		["has", {what: "output", asset: "base", address: "MO7ZZIU5VXHRZGGHVSZWLWL64IEND5K2"}],
		["has one equal", {
			equal_fields: ["amount"], 
			search_criteria: [{what: "output", asset: "base"}, {what: "output", asset: "this asset"}]
		}]
	]],
	transfer_condition: ["and", [
		["has one", {what: "output", asset: "this asset"}],
		["has one equal", {
			equal_fields: ["address", "amount"], 
			search_criteria: [{what: "output", asset: "base"}, {what: "output", asset: "this asset"}]
		}]
	]]
};
```

The above conditions stipulate that the newly defined asset can be issued if the issuer also sends equal amount of bytes to the address MO7ZZIU5VXHRZGGHVSZWLWL64IEND5K2, and it can be transferred if equal amount of bytes is transferred to the same address at the same time.

## Issuance of a new asset

The new asset can be issued after its definition is confirmed. To issue, you need just to spend the asset from an address that is allowed to issue and it will be issued to address you transferred it.

You cannot use `wallet.sendMultiPayment()`, `headlessWallet.issueChangeAddressAndSendPayment()`and similar functions because they can do only transfers and they don't know about addresses that can issue new coins. Instead, you have to use lower level functions:

* `composeAndSaveDivisibleAssetPaymentJoint` from `ocore/divisible_asset.js`
* `composeAndSaveIndivisibleAssetPaymentJoint` from `ocore/indivisible_asset.js`

See [create\_divisible\_asset\_payment.js](https://github.com/byteball/headless-obyte/blob/master/tools/create_divisible_asset_payment.js) \(assets without fixed denominations\) and [create\_indivisible\_asset\_payment.js](https://github.com/byteball/headless-obyte/blob/master/tools/create_indivisible_asset_payment.js) \(assets with fixed denominations\) for examples.

See an example of how an asset can be both defined and issued in a single script [https://github.com/byteball/ico-bot/blob/master/scripts/issue\_tokens.js](https://github.com/byteball/ico-bot/blob/master/scripts/issue_tokens.js)

## Whitepaper

For more details see chapter 24 of the [whitepaper](https://byteball.org/Byteball.pdf).

