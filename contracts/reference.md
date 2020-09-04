# Smart contract language reference

## Authentication

These clauses authenticate the author\(s\) of the unit.

### sig

Here is an example of the simplest address definition that defines an address controlled by a single private key:

```javascript
["sig", {"pubkey": "Ald9tkgiUZQQ1djpZgv2ez7xf1ZvYAsTLhudhvn0931w"}]
```

The pubkey above is base64-encoded public key. The `sig` expression evaluates to `true` if the signature provided with the transaction is valid and produced by the private key that corresponds to the above public key. The address \(checksummed hash in base32\) corresponding to this definition is A2WWHN7755YZVMXCBLMFWRSLKSZJN3FU.

### hash

This clause evaluates to `true` if the author supplies a preimage of the hash specified in the address definition.

```javascript
["hash", {"hash": "value of sha256 hash in base64"}]
```

## Logical operators

These clauses allow to combine other conditions with logical operators.

### and, or

All expressions in this language evaluate to a boolean value, and multiple boolean subexpressions can be combined using boolean operators `and` and `or`. For example, this is a definition that requires two signatures:

```javascript
["and", [
    ["sig", {pubkey: "one pubkey in base64"}],
    ["sig", {pubkey: "another pubkey in base64"}]
]]
```

To spend funds from the address equal to the hash of the above definition, one would need to provide two signatures.

As you noticed, we use JSON to construct the language expressions. This is an unusual choice but allows to use existing well-debugged, well-supported, and well-optimized JSON parsers rather than invent our own.

“Or” condition can be used to require signatures by any one of the listed public keys:

```javascript
["or", [
    ["sig", {pubkey: "laptop pubkey"}],
    ["sig", {pubkey: "smartphone pubkey"}],
    ["sig", {pubkey: "tablet pubkey"}]
]]
```

The above is useful when you want to control the same address from any of the 3 devices: your laptop, your phone, and your tablet.

The conditions can be nested:

```javascript
["and", [
    ["or", [
        ["sig", {pubkey: "laptop pubkey"}],
        ["sig", {pubkey: "tablet pubkey"}]
    ]],
    ["sig", {pubkey: "smartphone pubkey"}]
]]
```

### r of set

A definition can require a minimum number of conditions to be true out of a larger set, for example, a 2-of-3 signature:

```javascript
["r of set", {
    required: 2,
    set: [
        ["sig", {pubkey: "laptop pubkey"}],
        ["sig", {pubkey: "smartphone pubkey"}],
        ["sig", {pubkey: "tablet pubkey"}]
    ]
}]
```

\(“r” stands for “required”\) which features both the security of two mandatory signatures and the reliability, so that in case one of the keys is lost, the address is still usable and can be used to change its definition and replace the lost 3rd key with a new one, or to move the funds to another address.

### weighted and

Also, different conditions can be given different weights, of which a minimum is required:

```javascript
["weighted and", {
    required: 50,
    set: [
        {weight: 40, value: ["sig", {pubkey: "CEO pubkey"}] },
        {weight: 20, value: ["sig", {pubkey: "COO pubkey"}] },
        {weight: 20, value: ["sig", {pubkey: "CFO pubkey"}] },
        {weight: 20, value: ["sig", {pubkey: "CTO pubkey"}] }
    ]
}]
```

### not

Subsequent conditions can be negated with `not` clause:

```javascript
["not", ["in data feed", [["NOAA ADDRESS"], "wind_speed", ">", "200"]]]
```

Since it is legal to select very old parents \(that didn’t see the newer data feed posts\), one usually combines negative conditions such as the above with the requirement that the timestamp is after a certain date.

`sig`, `hash`, `address`, `cosigned by`, and `in merkle` cannot be negated.

## References

These clauses redirect the evaluation of the subdefinition to something else.

### address

A definition can contain reference to another address using `address` clause:

```javascript
["and", [
    ["address", "ADDRESS 1 IN BASE32"],
    ["address", "ADDRESS 2 IN BASE32"]
]]
```

which delegates signing to another address and is useful for building shared control addresses \(addresses controlled by several users in contracts\). This syntax gives the users the flexibility to change definitions of their own component addresses whenever they like, without bothering the other user.

### definition template

A definition can reference a definition template:

```javascript
["definition template", [
    "hash of unit where the template was defined",
    {param1: "value1", param2: "value2"}
]]
```

The parameters specify values of variables to be replaced in the template. The template needs to be saved before \(and as usual, be stable before use\) with a special message type app=“definition\_template”, the template itself is in message payload, and the template looks like normal definition but may include references to variables in the syntax @param1, @param2. Definition templates enable code reuse. They may in turn reference other templates.

## Extrospection

These clauses inspect data **outside** of the current unit.

### in data feed

`in data feed` clause can be used to make queries about data previously stored on Obyte:

```javascript
["in data feed", [
    ["ADDRESS1", "ADDRESS2", …], 
    "data feed name", 
    "=", 
    "expected value"
]]
```

This condition evaluates to `true` if there is at least one previous message stored in Obyte database that has “data feed name” equal to “expected value”. Instead of `=`, you can also use `>`, `<`, `>=`, `<=`, or `!=` The data feed must be posted to Obyte decentralized database by one of the oracles whose addresses are “ADDRESS1”, “ADDRESS2”, … Since oracles post to the common database, we call them on-chain oracles.

On-chain oracles are a very powerful thing indeed. For example, this address definition represents a binary option:

```javascript
["or", [
    ["and", [
        ["address", "ADDRESS 1"],
        ["in data feed", [["EXCHANGE ADDRESS"], "EURUSD", ">", "1.2500"]]
    ]],
    ["and", [
        ["address", "ADDRESS 2"],
        ["timestamp", [">", Math.round(Date.now()/1000 + timeout_seconds)]]
    ]]
]]
```

Initially, the two parties fund the address defined by this definition by sending their respective stakes to the address. Then if the EUR/USD exchange rate published by the exchange address ever exceeds 1.2500, the first party can sweep the funds. If this doesn’t happen before timeout period, the second party can sweep all the funds stored on this address.

Another example would be a customer who buys goods from a merchant but he doesn’t quite trust that merchant and wants his money back in case the goods are not delivered. The customer pays to a shared address defined by:

```javascript
["or", [
    ["and", [
        ["address", "MERCHANT ADDRESS"],
        ["in data feed", [["FEDEX ADDRESS"], "tracking", "=", "123456"]]
    ]],
    ["and", [
        ["address", "BUYER ADDRESS"],
        ["timestamp", [">", Math.round(Date.now()/1000 + timeout_seconds)]]
    ]]
]]
```

The definition depends on the FedEx oracle that posts tracking numbers of all successfully delivered shipments. If the shipment is delivered, the merchant will be able to unlock the money using the first condition. If it is not delivered before the specified timeout period, the customer can take his money back. This example is somewhat crazy because it requires FedEx to post each and every shipment. See `in merkle` clause below for a more practical way to achieve the same result.

### in merkle

`in merkle` is a more economical way to query the presence of a particular data entry in a large data set. Instead of posting every data entry in a `data_feed` message, only the merkle root of the large data set is posted as a `data_feed`, and the signer has to provide the data entry and its merkle path:

```javascript
["in merkle", [
    ["ADDRESS1", "ADDRESS2", ...],
    "data feed name",
    "expected value"
]]
```

### seen address

```javascript
["seen address", "ANOTHER ADDRESS IN BASE32"]
```

This clause evaluates to `true` if the specified address was seen as author in at least one past unit included in the last stable unit.

### seen

```javascript
["seen", {
    what: "output",
    address: "ADDRESS",
    asset: "asset or base",
    amount: 12345
}]
```

This clause evaluates to `true` if there was an input or output in the past \(before last stable unit\) that satisfies the specified condition. The syntax for the search condition is the same as for `has` clause below.

### seen definition change

```javascript
["seen definition change", ["ADDRESS", "NEW DEFINITION CHASH"] ]
```

This clause evaluates to `true` if there was a definition change of the specified address and the c-hash \(checksummed hash\) of the new definition is equal to the specified value.

### age

```javascript
["age", [">", 1234]]
```

This clause evaluates to `true` if the age of all inputs spent from this address satisfies the specified condition. The age is the difference between last ball mci and the input’s mci.

### attested

```javascript
["attested", ["ADDRESS", ["ATTESTOR1", "ATTESTOR2", ...]]]
```

This clause evaluates to `true` if the specified address is attested by one of the listed attestors. The address can also be “this address”.

## Introspection

These clauses inspect data **within** the current unit

### cosigned by

A subdefinition may require that the transaction be cosigned by another address:

```javascript
["cosigned by", "ANOTHER ADDRESS IN BASE32"]
```

### has, has one

A definition can also include queries about the transaction itself, which can be used for example to code limit orders on a trustless exchange. Assume that a user wants to buy 1,200 units of some asset for which he is willing to pay no more than 1,000 bytes \(the native currency of Obyte\). Also, he is not willing to stay online all the time while he is waiting for a seller. He would rather just post an order at an exchange and let it execute when a matching seller comes along. He can create a limit order by sending 1,000 bytes to an address defined by this definition, which makes use of `has` clause:

```javascript
["or", [
    ["address", "USER ADDRESS"],
    ["and", [
        ["address", "EXCHANGE ADDRESS"],
        ["has", {
            what: "output", 
            asset: "ID of alternative asset", 
            amount_at_least: 1200, 
            address: "USER ADDRESS"
        }]
    ]]
]]
```

The first or-alternative lets the user take back his bytes whenever he likes, thus cancelling the order. The second alternative delegates the exchange the right to spend the funds, provided that another output on the same transaction pays at least 1,200 units of the other asset to the user’s address. The exchange would publicly list the order, a seller would find it, compose a transaction that exchanges assets, and sign it together with the exchange. Note that the exchange does not receive arbitrary control over the user’s funds, it can spend them only if it simultaneously pays the alternative asset to the user, while the user retains full control over his funds and can withdraw them from the contract when he likes.

The `has` clause evaluates to `true` if the transaction has at least one input or output that satisfies all the set conditions:

* `what`: `input` or `output`, required
* `asset`: ID of asset \(44-character string\) or `base` for bytes
* `type`: for inputs only, `transfer` or `issue`. If specified, it searches only transfers or only issues
* `amount_at_least`, `amount_at_most`, `amount`: amount must be at least, at most, or exactly equal to the specified value
* `address`: address where the output is sent to or the input is spent from, can be literal address or “this address” or “other address”

Similar clause `has one` has exactly the same syntax but evaluates to true if there is exactly one input or output that satisfies the search conditions.

### has equal, has one equal

The following requirement can also be included in a subdefinition:

```javascript
["has equal", {
    equal_fields: ["address", "amount"],
    search_criteria: [
        {what: "output", asset: "asset1", address: "ADDRESS IN BASE32"},
        {what: "input", asset: "asset2", type: "issue", address: "ANOTHER ADDRESS IN BASE32"}
    ]
}]
```

It evaluates to `true` if there is at least one pair of inputs or outputs that satisfy the search criteria and the fields specified in `equal_fields` are equal. The first element of the pair is searched by the first set of filters, the second by the second, and the syntax for the search conditions is the same as in `has` clause.

A similar condition `has one equal` requires that there is exactly one such pair.

### sum

This clause evaluates to `true` if the sum of inputs or outputs that satisfy the filter is equal, at least, or at most the target value.

```javascript
["sum", {
    filter: {
        what: "input"|"output", asset: "asset or base", type: "transfer"|"issue",
        address: "ADDRESS IN BASE32"
    },
    at_least: 120,
    at_most: 130,
    equals: 123
}]
```

The syntax for the filter is the same as for `has` clause.

### has definition change

```javascript
["has definition change", ["ADDRESS", "NEW DEFINITION CHASH"] ]
```

This clause evaluates to `true` if the unit has a definition change of the specified address and the c-hash \(checksummed hash\) of the new definition is equal to the specified value.

### mci

```javascript
["mci", ">", 123456]
```

This clause evaluates to `true` if `last_ball_mci` of the current unit is greater than \(other possible comparisons: `>=`, `<`, `<=`, `=`\) than the specified value. It can be useful to make the address spendable only after some point in the future and not rely on any timestamp oracles.

### timestamp

```javascript
['timestamp', ['>', Math.round(Date.now() / 1000 + 2 * 3600)]]
```

This clause evaluates to `true` If the time is longer than the specified time. You can also use `"=", "<", ">=", "<=", "!="`. This example returns true 2 hours after the deployment.

