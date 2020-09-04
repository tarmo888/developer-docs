# Oscript language reference

## Autonomous agents

Autonomous agents \(AA\) are special addresses \(accounts\) on the ledger that do not belong to anybody. An AA can send transactions only in response to a triggering transaction sent to it and strictly according to a program associated with the AA. This program is open and known in advance, it specifies the AA's reaction to triggering transactions. The reaction depends on:

* coins sent by the triggering transaction: how much of which asset were sent;
* data sent by the triggering transaction;
* environment that exists in the ledger when the trigger is received: balances, data posted by oracles, attestations, state variables.

The AA's reaction can be one or a combination of:

* sending some other coins back to the sender or to a third party \(which can be another AA\);
* changing the environment of the ledger by posting data feeds, attestations, updating state variables, etc.

The behavior of autonomous agents is similar to vending machines: they accept coins and data entered on a keypad \(the triggering action\), and in response they release a cup of coffee or play a song, or do whatever they are programmed to do. What's common between them, their behavior is predictable, known in advance.

There are no private/public keys associated with AAs. Their transactions do not have signatures. Their transactions are created by all \(full\) nodes just by following the common protocol rules and executing the code associated with the AA. All nodes always come to the same result and produce exactly the same transaction that should be sent on behalf of the AA. As one comes to expect from a DAG, all nodes arrive at the same view of the ledger state just by following the rules, without any votings, proof-of competitions, or leaders.

The AA-generated transactions are not broadcast to peers as peers are expected to generate the same transactions on their own.

The language used to specify the AA behavior is a domain specific language that we call Oscript. It was designed specifically for AAs to make it easy to write an autonomous agent behavior, make it easy to argue about what is going on in the AA code, and make it hard to make difficult-to-track errors.

To control the resource consumption, the language does not support loops. Resource usage is controlled by setting a cap on the complexity of the program -- too resource heavy programs are simply not allowed. However, the resource caps provide enough room for the vast majority of practical applications.

AAs can "call" other AAs by sending coins to them. All the scripts in the calling AA are finished and all changes committed before passing control on to the next AA, this avoids completely the reentrancy problem common in other smart contract platforms.

## Autonomous agent definition

Addresses of autonomous agents follow the same general rules as all other Obyte addresses: their definitions are two-element arrays and the address is a checksummed hash of the array encoded in base32.

AA address is defined as:

```javascript
["autonomous agent", {
    // here goes the AA code
}]
```

The second element of the above array is an object that defines a template for future units created by the AA. The template's structure follows the structure of a regular unit in general, with some elements dynamic and dependent upon the input and state parameters. The dynamic elements are designated with special markup and include code in a domain specific language called Oscript:

```javascript
{
    address: "{trigger.address}",
    amount: "{trigger.output[[asset=base]] - 1000}"
}
```

The concept is similar to how such languages as PHP, ASP, and JSP work. For example, a PHP script is a HTML file interspersed with fragments of PHP code enclosed between `<?php` and `?>` tags. These fragments make the HTML page dynamic while keeping its HTML structure. While this mix can quickly become messy when PHP is used to generate HTML code, the ease of creating dynamic web pages by just inserting small pieces of code into HTML was one of the main selling points of these programming languages in the early days of the web. And it enabled quick prototyping, iteration, experimentation, and eventually led to the creation of some of the biggest pieces of the modern Internet \(wordpress and facebook, to name a few\).

Transactions, or storage units as we call them in Obyte, are similarly the basic building units of distributed ledgers. They are usually represented as JSON objects, which are the counterparts of HTML pages on the web. Now, to make it easy to create dynamic, parameterized JSON objects representing transactions, we allow to inject some code into JSON and invite the developers to apply their creativity and do the rest.

|  | Document format | Scripting language |
| :--- | :--- | :--- |
| Web | HTML | PHP |
| Obyte | JSON | Oscript |

This is an example of autonomous agent definition:

```javascript
["autonomous agent", {
    bounce_fees: { base: 10000 },
    doc_url: "https://example.com/doc_urls/{{aa_address}}.json",
    messages: [
        {
            app: "payment",
            payload: {
                asset: "base",
                outputs: [
                    {
                        address: "{trigger.address}",
                        amount: "{trigger.output[[asset=base]] - 1000}"
                    }
                ]
            }
        }
    ]
}]
```

Here `messages` is a template for the AA's response transaction. It has only one message -- a payment message that will send the received amount less 1000 bytes back to sender. Omitting the `amount` entirely would send everything back \(minus the fees\). There are code fragments in strings enclosed in curly braces "{}", they are evaluated and the result is inserted in place of the code.

Before sending the resulting response transaction, the nodes on the network will also enhance it with change outputs \(if necessary\) and add other necessary fields in order to attach the transaction to the DAG: parents, last stable unit, authors, timestamp, fees, etc.

Below is a specification of the unit template format.

## JSON format

### bounce\_fees

```javascript
["autonomous agent", {
    bounce_fees: {
        base: 10000,
        "n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY=": 100
    },
    ...
}]
```

This is an optional field of the unit template that specifies the fees charged from sender if the AA execution fails. In this case, all the received money in all assets is automatically bounced back to sender, less the bounce fees. The fees are keyed by asset ID \(`base` for bytes\).

The minimum and default bounce fee for bytes is 10000 bytes. The minimum and default bounce fee for all other assets is 0. Non-base bounce fees apply only to those assets that were actually received by the autonomous agent.

Sending to an autonomous agent anything less than the bounce fees will result in no response and the AA silently eating the coins. However this rule applies only to money sent from regular addresses. Bounce fees are not checked when the money is received from another AA.

`bounce_fees` field is removed from the final unit.

### doc\_url

Each deployed autonomous agent can have a URL that points to a JSON formatted documentation file, which content will be shown to users in wallet app.  URL can contain `{{aa_address}}` placeholder, which will be replaced with actual AA address before fetching the JSON file. The structure of the JSON file should look like this:

```javascript
{
	"version": "1.0",
	"description": "Description shown to users",
	"homepage_url": "https://example.com",
	"source_url": "https://github.com/byteball/ocore",
	"field_descriptions": {
		"some_field_name": "Description how to use this parameter",
		"some_other_field_name": "Description how to use this other parameter",
		...
	}
}
```

Keys in `field_descriptions` object should match all the keys in `trigger.data` that users can use with this AA, wallet app will add value of that fields as an explanation what it does. The value of `version` should be as shown above \(`"1.0"`\) - it should NOT used as version of AA.

### messages

This is the main field of autonomous agent definition. It specifies templates for the messages to be generated, and the templates are parameterized with oscript code.

The messages can be of any type \(called `app`\) that Obyte supports. The most common app is `payment`, it is used to send payments in any asset back to sender or to a third party. Other apps are:

* `asset`: used to define a new asset. Different parameters that can be used are documented on [Issuing assets page](../issuing-assets-on-byteball.md#defining-a-new-asset);
* `data`: used to send data, this includes sending data parameters to other \(secondary\) AAs;
* `data_feed`: used to send data feeds.  By doing this, the AA becomes an oracle;
* `profile`: used to send one's own profile.  Maybe an AA wants to say something to the world about itself;
* `text`: used to save arbitrary text to the DAG;
* `definition`: used to post a definition of a new AA;
* `asset_attestors`: used to change the attestor list of an asset previously defined by this AA;
* `attestation`: used to post information about some other address.  By doing this, the AA becomes an attestor;
* `definition_template`: used to post a template for smart contract definition;
* `poll`: used to create a poll;
* `vote`: used to vote in a poll.  Every AA has voting rights after all.

Structure of those messages types is documented on [Sending data messages section](getting-started-guide.md#sending-data-messages) and [Sending data to DAG page](../payments/data.md).

There is also another, special, app called `state`, which is not possible in regular units but is used only in AAs to produce state changes. More about it in a separate chapter.

### base\_aa

It is possible to create parameterized Autonomous Agents, which are based on previously deployed templates. Their structure for that kind of Autonomous Agent would look like this:

```javascript
["autonomous agent", {
    base_aa: "ADDRESS_OF_BASE_AA",
    params: {
        name1: "value1",
        name2: "value2",
        ...
    }
}]
```

The base AA template would need to reference these parameters as `params.name1` and `params.name2`.

## Oscript replacements

Any string, number, or boolean in the template can be calculated by a script. The script is evaluated and the result is inserted in place of the script, the type is preserved. For example, if 20000 bytes were sent from address `2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7` to an AA, then this code

```javascript
{
    address: "{trigger.address}",
    amount: "{trigger.output[[asset=base]] - 1000}"
}
```

would be replaced with

```javascript
{
    address: "2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7",
    amount: 19000
}
```

Object keys can also be parameterized with Oscript:

```javascript
{
    "{trigger.data.key}": "value"
}
```

See the Oscript language reference below.

### cases - template objects

JSON has scalars \(strings, numbers, booleans\) and objects \(objects and arrays\). Scalars can be parameterized through Oscript. Objects, on the other hand, are parameterized differently. They can have several alternative versions in the AA template, and only one version is selected based on input parameters and state. The versions and their selection criteria are specified using an object in `cases` array:

```javascript
{
    messages: {
        cases: [
            {
                if: "{trigger.data.define}",
                messages: [
                    // first version of messages
                    ....
                ]
            },
            {
                if: "{trigger.data.issue}",
                init: "{$amount = trigger.output[[asset=base]];}",
                messages: [
                    // second version of messages
                    ....
                ]
            },
            {
                messages: [
                    // default version of messages
                    ....
                ]
            }
        ]
    }
}
```

The regular value of an object/array is replaced with an object whose single element is an array `cases`. Each element of the `cases` array is an object with up to 3 elements:

* `if`: an Oscript expression.  If the result of its evaluation is truthy then this `case` is selected.  All other `case`s are not evaluated.  `if` is required for all `case`s except the last, the last one may or may not have an `if`.  If all previous `case`s evaluated to a falsy value and the last one is without an `if`, the last one is selected;
* `init`: an optional statements-only Oscript that is evaluated immediately after `if` if this `case` is selected;
* a mandatory element that is named the same as the original field \(`messages` in the above example\).  If this `case` is selected, the original \(3 levels higher\) field is replaced with the value of this element.

In the above example, if the 2nd `case` were selected, the original object would fold into:

```javascript
{
    messages: [
        // second version of messages
        ....
    ]
}
```

Cases can be nested.

Cases can be used for any non-scalar value inside `messages`, not just `messages` themselves.

### if - conditional objects

Similar to the `cases` above, any object can have an additional `if` field. It is evaluated first, and if it is falsy, the entire object is removed from the enclosing object or array. Its internal Oscripts are not evaluated in this case.

```javascript
{
    messages: [
        {
            app: "data",
            payload: {
                timestamp: "{timestamp}",
                subscriber: "{trigger.address}"
            }
        },
        {
            if: "{trigger.data.withdrawal_amount > 0}",
            app: "payment",
            payload: {
                asset: "base",
                outputs: [
                    {
                        address: "{trigger.address}",
                        amount: "{trigger.data.withdrawal_amount}"
                    }
                ]
            }
        }
   ]
}
```

In the above example, the `payment` message will be generated only if `trigger.data.withdrawal_amount` is a number greater than 0.

The `if` field itself is removed from the object.

### init - statements object

Similar to the `cases` above, any object can have an additional `init` field. It is evaluated immediately after `if` when `if` is present and truthy. If there is no `if`, `init` is unconditionally evaluated first.

`init` must be a statements-only Oscript, it does not return a value.

Example:

```javascript
{
    messages: [
        {
            init: "{ $addr = trigger.address; }",
            app: "data",
            payload: {
                timestamp: "{timestamp}",
                subscriber: "{$addr}"
            }
        },
        {
            if: "{trigger.data.withdrawal_amount > 1000}",
            init: "{ $amount = trigger.data.withdrawal_amount - 1000; }",
            app: "payment",
            payload: {
                asset: "base",
                outputs: [
                    {
                        address: "{trigger.address}",
                        amount: "{$amount}"
                    }
                ]
            }
        }
   ]
}
```

The `init` field itself is removed from the object.

### Conditional object fields and array elements

If the value of any object field or array value evaluates to an empty string, this field or element is removed.

For example, object

```javascript
{
    field1: "{ (1 == 2) ? 'value1' : '' }",
    field2: "value2"
}
```

will become

```javascript
{
    field2: "value2"
}
```

Array

```javascript
[ `{ (1 == 2) ? "value1" : "" }`, "value2" ]
```

will become

```javascript
[ "value2" ]
```

### Sending all coins

If the `amount` field in an output within a payment message is omitted or evaluates to an empty string \(which results in its removal per the above rules\), this output receives all the remaining coins.

```javascript
{
    messages: [
        {
            if: "{trigger.data.send_all}",
            app: "payment",
            payload: {
                asset: "base",
                outputs: [
                    {
                        address: "2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7",
                        amount: 1000
                    },
                    {
                        address: "{trigger.address}"
                    }
                ]
            }
        }
   ]
}
```

In the above example, 1000 bytes are sent to `2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7`, the rest of the AA's balance in bytes is sent to the address that triggered the AA.

### Empty objects and arrays

If an object or array becomes empty as a result of various removals, it is also removed from the enclosing object or array.

### State message

A state message is a special message in the `messages` array that performs state changes. It is the only oscript where state variables are assigned. Unlike regular messages that always have `payload`, state message has a field named `state` instead that contains a state changing script:

```javascript
{
    messages: [
        {
            app: "payment",
            payload: {
                asset: "base",
                outputs: [
                    {
                        address: "{trigger.address}",
                        amount: "{trigger.output[[asset=base]] - 1000}"
                    }
                ]
            }
        },
        {
            app: "state",
            state: `{
                var['responded'] = 1;
                var['total_balance_sent_back'] += trigger.output[[asset=base]] - 1000;
                var[trigger.address || '_response_unit'] = response_unit;
            }`
        }
    ]
}
```

The state message must always be the last message in the `messages` array. It is not included in the final response unit and its script \(state script\) is evaluated **after** the response unit is already prepared. It is the only oscript where response\_unit variable is available. State script contains only statements, it is not allowed to return any value.

## Constants/Variables

### Local constants

```php
$name1 = 1;
$name2 = 'value';
${'name' || 3} = $name1 + 10;
```

Local constant names are always prefixed with `$`. If the constant name itself needs to be calculated, the expression is enclosed in curly braces.

Local constants exist only during AA evaluation. Each constant can be assigned a value only once \(single-assignment rule\).

Each local constant is visible only in its own oscript after it was assigned. If it was assigned in `if` block, it is also available in all adjacent and enclosed oscripts. If it was assigned in `init` block, it is also available in all adjacent oscripts except `if` and in all oscripts enclosed in the same object.

```javascript
[
    {
        if: `{
            $amount = trigger.output[[asset=base]];
            // the result of the last expression is the result of if
            $amount == 10000
        }`,
        init: `{
            // here we can reference $amount set in if
            $half_amount = round($amount / 2); 
        }`,
        messages: [
            {
                app: "payment",
                payload: {
                    asset: "base",
                    outputs: [
                        {
                            address: "{ trigger.address }",
                            amount: `{
                                // we are in an enclosed oscript
                                // and can reference $half_amount set in init
                                $half_amount
                            }`
                        }
                    ]
                }
            },
            {
                app: "state",
                state: `{
                    // we are in an enclosed oscript
                    // and can reference $amount set in if
                    var['received'] = $amount;
                    // we are in an enclosed oscript
                    // and can reference $half_amount set in init
                    var['sent_back'] = $half_amount;
                }`
            }
        ]
    },
    {
        if: "{trigger.data.payout}",
        init: `{
            // here we cannot reference $amount nor $half_amount from the above
            // we can even assign other values to them without breaking the single-assignment rule
            $amount = 10;
        }`,
        ...
    }
]
```

If an unassigned local constant is referenced, it is taken as `false`.

If-else blocks in curly braces do not create separate scopes for local constants:

```javascript
if (trigger.data.deposit){
    $amount = trigger.output[[asset=base]];
}
$fee = round($amount * 0.01); // we can reference the $amount from above
```

Local constants can hold values of any type: string, number, boolean, or object \(including array\).

### Objects and arrays

When a local constant holds an object, individual fields of the object can be accessed through dot or \[\] selector:

```javascript
$data = trigger.data;
$action = $data.params.action;
$player_score = $data.params[$player_name || '_score'];
```

If the specified object field does not exist, the value is taken as `false`.

Objects and arrays can be initialized using familiar syntax as in other languages:

```javascript
$obj = {a: 3, b: 7 };
$arr = [7, 2, 's', {a: 6}];
```

Although all local constants are single-assignment, objects and arrays can be mutated by modifying, adding, or deleting their fields:

```javascript
$obj = {a: 3, b: 7 };
$obj.a = 4; // modifies an existing field
$obj.c = 10; // adds a new field
delete($obj, 'b'); // deletes a field
freeze($obj); // prohibits further mutations of the object

$arr = [7, 2, 's', {a: 6}];
$arr[0] = 8; // modifies an existing element
$arr[] = 5; // adds a new element
delete($arr, 1); // removes element 1
freeze($arr); // prohibits further mutations of the array
```

The left-hand selectors can be arbitrarily long `.a.b[2].c.3.d[].e`. If some elements do not exist, an empty object or array is created automatically. `[]` adds to the end of an array. Array indexes cannot be skipped, i.e. if an array has length 5, you cannot assign to element 7. Once you are dome mutating an object, can call `freeze()` to prevent further accidental mutations.

### Local functions

```javascript
$f = ($x) => {
	 $a = var['a'];
	 $x * $a
};

// one-line syntax for functions that have only one expression
$sq = $x => $x^2;

$res = $f(2);
$nine = $sq(3);
```

Functions are local constants and the same rules apply to them.

The return value is the value of the last expression or a return statement.

A function sees all other local constants and functions declared _before_ it. Because of this, recursion is impossible.

Constants declared before the function cannot be shadowed by its arguments or other local constants declared within the function.

Complexity of a function is the sum of complexities of all operations within it. Every time a function is called, its complexity is added to the total complexity of the AA. If a function is declared but never called, its complexity doesn't affect the complexity of the AA.

### Remote functions \(getters\)

Getters are read-only functions available to other AAs and non-AA code.

They are meant to extract information about an AA state that is not directly available through state vars. E.g. in order to fetch this information one needs to perform some calculation over state vars or access several state vars and have good knowledge about the way information is stored in the AA.

**Examples:**

* oswap could expose functions that would calculate the price of a future exchange. Otherwise, clients would have to do non-trivial math themselves.
* the token registry could expose a single getter `$getTokenDescription($asset)` for reading a token description, and a similar one for decimals. Otherwise, one has to read first `$desc_hash = var['current_desc_' || $asset]`, then `var['description_' || $desc_hash]`.

Getters are declared in a top-level `getters` section which is evaluated before everything else.

```javascript
['autonomous agent', {
	getters: `{
		$sq = $x => $x^2;
		$g = ($x, $y) => $x + 2*$y;
		$h = ($x, $y) => $x||$y;
		$r = ($acc, $x) => $x + $acc;
	}`,
	init: `{
		// uncomment if the AA serves as library only
		// bounce("library only");
		...
	}`,
	...
}]
```

The code in `getters` section can contain only function declarations and constants. Request-specific information such as `trigger.data`, `trigger.outputs`, etc is not available in getters.

In the AA which declares them, getters can be accessed like normal functions.

Other AAs can call them by specifying the remote AA address before the function name using this syntax:

```javascript
$nine = MXMEKGN37H5QO2AWHT7XRG6LHJVVTAWU.$sq(3);
```

or

```javascript
$remote_aa = "MXMEKGN37H5QO2AWHT7XRG6LHJVVTAWU";
$nine = $remote_aa.$sq(3);
```

where `$remote_aa` variable must be a constant \(it must be known at deploy time in order calculate the complexity\).

The complexity of a remote call is the complexity of its function, plus one.

All functions declared in the `getters` section are publicly available. If some of them are not meant for public use, one can indicate this by a naming convention, e.g. by starting their names with an underscore `$_callPrivateFunction()` but information hiding cannot be really enforced since all getters operate on public data anyway.

Getters can also be conveniently called from non-AAs. In node.js code:

```javascript
const { executeGetter } = require('ocore/formula/evaluation.js');
const db = require('ocore/db.js');

const args = ["arg1", "arg2"];
const res = await executeGetter(db, aa_address, getter, args);
```

For remote clients, there is a `light/execute_getter` command in WebSocket API, hopefully it will be shortly available through obyte.js.

### State variables

State variables are persisted across invocations of autonomous agents.

Accessing state variables:

```javascript
// assigning current AAs state variable to local constant
$my_var_name1 = var['var_name1'];

// assigning other AAs state variable to local constant
$their_var_name1 = var['JVUJQ7OPBJ7ZLZ57TTNFJIC3EW7AE2RY']['var_name1'];
```

Assigning state variables:

```javascript
var['var_name1'] = 'var_value';
var['var_name2'] = 10;
var['var_name3'] += 10;
var['var_name4'] = false;
var['var_name5'] = {a:8, b:2};
var['var_name5'] ||= {c:6};  // concat an object
```

`var['var_name']` reads the value of state variable `var_name` stored under current AA.

`var['AA_ADDRESS']['var_name']` reads the value of state variable `var_name` stored under AA `AA_ADDRESS`. `AA_ADDRESS` is a valid address or `this_address` to refer to the current AA.

If there is no such variable, `false` is returned.

State variables can be accessed in any oscripts, but can be assigned only in [state message](oscript-language-reference.md#state-message) script. Only state vars of the current AA can be assigned, state vars of other AAs are read-only. State vars can be reassigned multiple times but only the final value will be saved to the database and only if the AA finishes successfully. All changes are committed atomically. If the AA fails, all changes to state vars are rolled back.

State vars can temporarily hold strings, numbers, objects, and booleans but when persisting, `true` values are converted to 1 and `false` values result in removal of the state variable from storage. AAs are required to hold a minimum balance in bytes equal to the size of their storage \(length of var names + length of var values\). This will also incentivize them to free up unused storage. The size of AA’s storage occupied before the current invocation is in variable `storage_size`.

Internally, objects are stored as JSON strings and their length is limited. Don't try to store a structure in a state var if this structure can grow indefinitely.

In addition to regular assignment `=`, state variables can also be modified in place using the following operators:

* `+=`: increment by;
* `-=`: decrement by;
* `*=`: multiply by;
* `/=`: divide by;
* `%=`: remainder of division by;
* `||=`: concatenate string/object/array.

For concatenation, the existing value of the var is converted to string.

For `+=`, `-=`, `*=`, `/=`, `%=`, the existing boolean value is converted to 1 or 0, strings result in error.

If the variable didn't exist prior to one of these assignments, it is taken as `false` and converted to number or string accordingly.

Each read or write operation on a state variable adds +1 to complexity. Assignment with modification also costs 1 in complexity.

Examples:

```javascript
var['sent_back'] = $half_amount;
var['count_investors'] += 1;
var['amount_owed'] += trigger.output[[asset=base]];
var['pending'] = false;
$x = var['JVUJQ7OPBJ7ZLZ57TTNFJIC3EW7AE2RY']['var_name1'];
```

### Response variables

```javascript
response['key'] = 'text';
```

Adds a key to the response object. Response variables do not affect state, they are meant to only inform the caller, and other interested parties, about the actions performed by the AA.

Response vars can only be assigned, never read. Response vars can be assigned and reassigned multiple times in any oscript. They can hold values of types: string, number, boolean. Attempting to assign an object would result in `true` being assigned.

Example: assigning these response variables

```javascript
response['message'] = "set exchange rate to 0.123 tokens/byte";
response['deposit'] = 2250000;
```

will result in the following response object:

```javascript
{
    "responseVars": {
        "message": "set exchange rate to 0.123 tokens/byte",
        "deposit": 2250000
    }
}
```

## Responses

The AAs are activated and responses are generated when the triggering unit gets stabilized. If the triggering unit triggers several AAs or there are several triggers that are included in the same MC unit and therefore get stabilized at the same time, the triggers are handled in deterministic order to ensure reproducibility on all nodes.

The first response unit has one or two parents:

* the MC unit that just got stabilized and which includes the triggering unit;
* the previous AA-generated unit \(meaning, generated by any AA, not just the current one\) if it is not already included in the first parent.

Any subsequent responses \(generated by secondary AAs and in response to other triggers at the same MCI\) are chained after the first response and then one after another.

After every response, 4 events are emitted:

* `aa_response`
* `aa_response_to_unit-` + trigger\_unit
* `aa_response_to_address-` + trigger\_address
* `aa_response_from_aa-` + aa\_address

Applications, which are based on a full node can subscribe to these events to receive information about responses they are interested in, e.g.:

```javascript
var trigger_address = '';

const eventBus = require('ocore/event_bus.js');
eventBus.on('aa_response_to_address-' + trigger_address, (objAAResponse) => {
    // handle event
});
```

Applications, which are based on light node will also need to add the address to their watched list in order to subscribe to these events:

```javascript
var aa_address = '';

const walletGeneral = require('ocore/wallet_general.js');
const eventBus = require('ocore/event_bus.js');
walletGeneral.addWatchedAddress(aa_address, () => {
    eventBus.on('aa_response_from_aa-' + aa_address, (objAAResponse) => {
        // handle event
    });
});
```

All 4 event handlers receive `objAAResponse` object as a single argument:

```javascript
{ 
    mci: 2385,
    trigger_address: '2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7',
    trigger_unit: 'f2S6Q3ufjzDyl9YcB51JUj2z9nE1sL4XL2VoYOrVRgQ=',
    aa_address: 'JVUJQ7OPBJ7ZLZ57TTNFJIC3EW7AE2RY',
    bounced: false,
    response_unit: 'JCJ1ZGkl2BtUlYoeu6U2yshp97pen/fIkTHvaKYjZa4=',
    objResponseUnit: {
        version: '2.0dev',
        alt: '3',
        timestamp: 1560939440,
        messages: [ ... ],
        authors: [ ... ],
        last_ball_unit: 'cxjDfHzWWgW8LqsC8yoDhgCYmwThmFfkdygGnDrxiFg=',
        last_ball: 'gjg8W2cE4WGFAIIsU2BvLOQKOpH/03Oo1SDS3/2SQDs=',
        witness_list_unit: '3gLI9EnI2xe3WJVPwRg8s4CB24ruetuddS0wYa2EI3c=',
        parent_units: [ 'f2S6Q3ufjzDyl9YcB51JUj2z9nE1sL4XL2VoYOrVRgQ=' ],
        headers_commission: 267,
        payload_commission: 157,
        unit: 'JCJ1ZGkl2BtUlYoeu6U2yshp97pen/fIkTHvaKYjZa4='
    },
    response: {} 
}
```

The object has the following fields:

* `mci`: the MCI of the triggering unit.  The response unit is attached to the NC unit with the same MCI;
* `trigger_address`: the address that sent coins to the AA and thus triggered it;
* `aa_address`: AA address
* `bounced`: `true` if the trigger was bounced, `false` otherwise;
* `response_unit`: hash of the response unit or null if there is no response;
* `objResponseUnit`: response unit object or null if there is no response;
* `response`: response from the script.  The object can have up to two fields: `error` for error message and `responseVars` for response variables set by scripts.

## Secondary AAs

The generated response unit may contain outputs to other AAs. In this case, the receiving AAs are triggered too. They receive the outputs from the previous AA and the data \(if any\) from its generated `data` message.

Secondary AAs behave like the primary ones except that they may receive less than the minimum bounce fees.

If there are several secondary AAs triggered by a previous AA, they are handled in a deterministic order to make sure that the results are reproducible on all nodes. If a secondary AA triggers a ternary one in turn, it is handled before going on to the next secondary AA.

If any of the secondary AAs fails, the entire chain fails, all the changes produced by its AAs are rolled back, and the primary AA bounces.

The total number of secondary AAs stemming from a single primary AA cannot exceed 10, otherwise the primary AA fails and bounces.

## Failures

If an AA fails for any reason \(bad formula, attempt to send an invalid response, attempt to send more money than it has, etc\), and attempt is made to bounce all the received coins back to sender, less bounce fees. All state changes are rolled back.

If creation of the bouncing transaction fails too, no transaction is created in response, i.e. the AA eats the coins.

The response object, which is not saved to the ledger, contains an error message explaining the reasons for failure.

## Semicolon

Every variable assignment must be terminated by a semicolon `;`. return and bounce statements are also terminated with a semicolon.

```javascript
$amount = trigger.output[[asset=base]];
var['sent_back'] = round($half_amount/2);
response['message'] = "set exchange rate to 0.123 tokens/byte";
if ($amount >= 42000)
    bounce("amount too large");
if (balance[base] < 20000)
    return;
```

## Two types of Oscripts

There are two types of Oscripts that can be used in AAs:

* statements-only scripts that consist only of statements \(such as assignments\) and don't return any value. init script and state message are the only two allowed statements-only scripts;
* scripts that return a value.  They can have 0 or more statements but the last evaluated expression must not be a statement and its result is the result of the script.

Example of a statements-only script:

```javascript
$amount = trigger.output[[asset=base]];
$action = trigger.data.action;
```

Example of a script that returns a value:

```javascript
$amount = trigger.output[[asset=base]];
$action = trigger.data.action;
$amount*2
```

The result of evaluation of the last expression `$amount*2` is returned.

### Non-scalar return values

Usually, the return value of oscript is a scalar: string, number, or boolean. It is just inserted in place of the oscript.

If the return value is an object, it is similarly expanded and inserted in place of the original oscript. This can be used to send prepared objects through `trigger.data`.

For example, if `trigger.data` is

```javascript
{
    output: {address: "BSPVULUCOVCNXQERIHIBUDLD7TIBIUHU", amount: 2e5}
}
```

and an AA has this message

```javascript
{
    app: "payment",
    payload: {
        asset: "base",
        outputs: [
            `{trigger.data.output}`
        ]
    }
}
```

The resulting message will be

```javascript
{
    app: "payment",
    payload: {
        asset: "base",
        outputs: [
            {
                address: "BSPVULUCOVCNXQERIHIBUDLD7TIBIUHU",
                amount: 2e5
            }
        ]
    }
}
```

## Flow control

### return

```javascript
return expr;
```

Interrupts the script's execution and returns the value of `expr`. This syntax can be used only in oscripts that are supposed to return a value.

```javascript
return;
```

Interrupts the script's execution without returning any value. This syntax can be used only in statements-only oscripts: `init` and `state`.

### if else

```javascript
if (condition){
    $x = 1;
    $y = 2 * $x;
}
else{
    $x = 2;
    $z = $x^3;
}
```

Evaluates the first block of statements if the `condition` is truthy, the second block otherwise. The `else` part is optional.

If the block includes only one statement, enclosing it in {} is optional:

```javascript
if (condition)
    $x = 1;
```

## Complexity

Like other smart contract definitions, AAs have a capped complexity which cannot exceed 100. Some operations involve complex computation or access to the database, such operations are counted and add to the total complexity count. Other operations such as `+`, `-`, etc, are relatively inexpensive and do not add to the complexity count. The language reference indicates which operations count towards complexity.

Total complexity is the sum of complexities of all oscripts. It is calculated and checked only during deployment and includes the complexity of all branches even though some of them might not be activated at run time.

If the complexity exceeds 100, validation fails and the AA cannot be deployed.

## Operators

#### Arithmetic operators +, -, \*, /, %, ^

```javascript
$amount - 1000
$amount^2 / 4
```

Operands are numbers or converted to numbers if possible.

Additional rules for power operator `^`:

* `e^x` is calculated with exact \(not rounded\) value of `e`,
* an exponent greater or equal to MAX\_SAFE\_INTEGER will cause an error,
* for non-integer exponents, the result is calculated as `x^y = e^(y * ln(x))` with rounding of the intermediary result, which causes precision loss but guaranties reproducible results;
* this operator adds +1 to complexity count.

### Concatenation operator \|\|

```javascript
'abc' || 'def'              // 'abcdef'
[4, 6] || [3, 1]            // [4, 6, 3, 1]
{x: 1, y: 7} || {y: 8, a:9} // {x: 1, y: 8, a:9}
```

If the same key is found in both objects, the value from the right-hand one prevails.

Trying to concat an array with object results in error.

If either operand is a scalar \(strings, numbers, booleans\), both are converted to strings and concatenated as strings. Objects/arrays become "true".

### Binary logical operators AND, OR

Lowercase names `and`, `or` are also allowed.

Non-boolean operands are converted to booleans.

The result is a boolean.

If the first operand evaluates to `true`, second operand of `OR` is not evaluated.

If the first operand evaluates to `false`, second operand of `AND` is not evaluated.

### Unary logical operator NOT

Lowercase name `not` is also allowed. The operator can be also written as `!`.

Non-boolean operand is converted to boolean.

The result is a boolean.

### Operator OTHERWISE

Lowercase name `otherwise` is also allowed.

```javascript
expr1 OTHERWISE expr2
```

If `expr1` is truthy, its result is returned and `expr2` is not evaluated. Otherwise, `expr2` is evaluated and its result returned.

### Comparison operators ==, !=, &gt;, &gt;=, &lt;, &lt;=

If both operands are booleans or both operands are numbers, the result is straightforward.

If both operands are strings, they are compared in lexicographical order.

If both operands are objects, only `==` and `!=` are allowed, other operators will cause an error.

If operands are of different types:

* if any of them is an object or boolean, it causes an error,
* if any of them is a string, only `==` and `!=` are allowed and non-string operand will be converted to string before comparison, other operators will cause an error,
* all other combinations of types cause an error.

### Ternary operator ? :

```javascript
condition ? expr1 : expr2
```

If `condition` is truthy, `expr1` is evaluated and returned, otherwise `expr2` is evaluated and returned.

## Operator precedence

Operators have the following precedence in the order of decreasing "stickiness":

* `^`
* `!`
* `*`, `/`, `%`
* `+`, `-`, `||`
* `==`, `!=`, `>`, `>=`, `<`, `<=`
* `AND`
* `OR`
* `?:`
* `OTHERWISE`

## Global constants

#### pi

Pi constant rounded to 15 digits precision: `3.14159265358979`.

#### e

Euler's number rounded to 15 digits precision: `2.71828182845905`.

## Built-in functions

### Type of variable

```javascript
typeof(anything)
```

Returns `"string"`, `"number"`, `"boolean"` or `"object"`.

### Square root and natural logarithm

```javascript
sqrt(number)
ln(number)
```

These functions add +1 to complexity count.

Negative numbers cause an error. Non-number inputs are converted to numbers or result in error.

### Absolute value

```javascript
abs(number)
```

Returns absolute value of a number. Non-number inputs are converted to numbers or result in error.

### Rounding \(round, ceil, floor\)

```javascript
round(number [, decimal_places])
ceil(number [, decimal_places])
floor(number [, decimal_places])
```

Rounds the input number to the specified number of decimal places \(0 if omitted\). `round` uses `ROUND_HALF_EVEN` rules. Non-number inputs are converted to numbers or result in error. Negative or non-integer `decimal_places` results in error. `decimal_places` greater than 15 results in error.

### Minimum and maximum

```javascript
min(number1, [number2[, number3[, ...]]])
max(number1, [number2[, number3[, ...]]])
```

Returns minimum or maximum among the set of numbers. Non-number inputs are converted to numbers or result in error.

### Square root of the sum of squares

```javascript
hypot(number1, [number2[, number3[, ...]]])
```

Returns the square root of the sum of squares of all arguments. Boolean parameters are converted to 1 and 0, objects are taken as 1, all other types result in error. The function returns a non-infinity result even if some intermediary results \(squares\) would overflow.

This function adds +1 to complexity count.

### Get part of a string

```javascript
substring(string, start_index)
substring(string, start_index, length)
```

Returns part of the string. If length is not set then returns rest of the string from start index. If `start_index` is negative then `substring` uses it as a character index from the end of the string. If `start_index` is negative and absolute of `start_index` is larger than the length of the string then `substring` uses 0 as the `start_index`.

### Find starting index of searched string

```javascript
index_of(string, search_string)
```

Returns integer index \(starting from 0\) of searched string position in string. If searched string is not found then -1 is returned. Use `contains` if you don't need to know the index of the searched string.

### String search within string

```javascript
starts_with(string, prefix)
ends_with(string, suffix)
contains(string, search_string)
```

Returns boolean `true` if the string starts, ends or contains searched string.

### Uppercase/Lowercase string

```javascript
to_upper(string)
to_lower(string)
```

Returns the string with changed case.

### Replace string

```javascript
replace(str, search_str, replacement)
```

Replaces all occurrences of `search_str` in `str` with `replacement` and returns the new string.

### Validate string

```javascript
has_only(str, allowed_chars)
```

Returns `true` if `str` consists only of characters in `allowed_chars`. `allowed_chars` is a group of characters recognized by regular expressions, examples: `a-z0-9`, `\w`. `has_only` adds +1 to complexity.

### Seconds until specified date

```javascript
parse_date(ISO8601_date)
parse_date(ISO8601_datetime)
```

Attempts to parse string of date or date + time and returns timestamp. If you need to get seconds from UNIX Epoch of a current unit then use [timestamp](oscript-language-reference.md#timestamp).

### Convert seconds to date + time, date or time

```javascript
timestamp_to_string(timestamp)
timestamp_to_string(timestamp, 'datetime')
timestamp_to_string(timestamp, 'date')
timestamp_to_string(timestamp, 'time')
```

Returns string format of date + time \(default\), date or time from [timestamp](oscript-language-reference.md#timestamp). Timezone is UTC.

### Parse a JSON string into object

```javascript
json_parse(string)
```

Attempts to parse the input JSON string. If the result of parsing is an object, the object is returned. If the result is a scalar \(boolean, string, number\), the scalar is returned.

This function adds +1 to complexity count.

If parsing fails, `false` is returned.

Non-string input is converted to string.

### Serialize an object into JSON string

```javascript
json_stringify(object)
```

Stringifies the input parameter into JSON. The parameter can also be a number, boolean, or string. If it is a number outside the IEEE754 range, the formula fails. Objects in the returned JSON are sorted by keys.

### Iteration methods \(map, reduce, foreach, filter\)

```javascript
$ar = [2, 5, 9];
$ar2 = map($ar, 3, $x => $x^2);
```

A function is executed over each element of an array or object. The callback function can be anonymous like in the example above, or referenced by name:

```javascript
$f = $x => $x^2;
$ar = [2, 5, 9];
$ar2 = map($ar, 3, $f);
```

It can also be a remote getter:

```javascript
$ar2 = map($ar, 3, $remote_aa.$f);
```

The function for map, foreach, and filter accepts 1 or 2 arguments. If it accepts 1 argument, the value of each element is passed to it. If it accepts 2 arguments, key and value for objects or index and elememt for arrays are passed.

The second argument is the maximum number of elements that an array or object can have. If it is larger, the script fails. This number must be a constant so that it can be known at deploy time, and the complexity of the entire operation is the complexity of the callback function times maximum number of elements. If the function has 0 complexity, the total complexity of map/reduce/foreach/filter is assumed to be 1 independently of the max number of elements. Max number of elements cannot exceed 100.

`reduce` has one additional argument for initial value:

```javascript
$c = 3;
$ar = [2, 5, 9];

// sums all elements, will return 16
$acc = reduce($ar, $c, ($acc, $x) => $acc + $x, 0);
```

The callback function for `reduce` accepts 2 or 3 arguments: accumulator and value or accumulator, key, and value \(accumulator, index, and element for arrays\).

These functions are similar to their Javascript counterparts but unlike Javascript they can also operate on objects, not just arrays.

### split, join

```javascript
split("let-there-be-light", "-")  // ["let", "there", "be", "light"]
join(["let", "there", "be", "light"], "-")  // "let-there-be-light"

split("let-there-be-light", "-", 2)  // ["let", "there"]
```

The functions are similar to their counterparts in other languages. `join` can be applied to objects too, in this case the elements are sorted by key and their values are joined.

### Reverse array

```javascript
reverse([4, 8, 3])  // [3, 8, 4]
```

The function reverses an array and returns a new one. Deep copies of all elements are created.

Passing a non-array to this function results in error.

### Keys of an object

```javascript
keys({b: 3, a: 8}) // ['a', 'b']
```

Returns the keys of an object. The keys are sorted. Passing anything but an object results in error.

### Length of any variable

```javascript
length(string|number|object|array)
```

Returns the length of string. number, object or array. When passed an object or array, it returns the number of elements in object or array. Scalar types are converted to strings and the length of the string is returned.

### Length of an array

```javascript
array_length(object)
```

Returns number of elements if the object is an array. Have to use [`is_array`](oscript-language-reference.md#is_array) to determine if object is an array. Use [`length`](oscript-language-reference.md#length-of-any-variable) instead.

### Number from seed string

```javascript
number_from_seed(string)
number_from_seed(string, max)
number_from_seed(string, min, max)
```

Generates a number from a seed string. The same seed always produces the same number. The numbers generated from different seed strings are uniformly distributed in the specified interval.

The first form returns a fractional number from 0 to 1.

The second form returns an integer number from 0 to max inclusive.

The third form returns an integer number from min to max inclusive.

This function is useful for generating pseudo-random numbers from a seed string. It adds +1 to complexity count.

### SHA-256 hash

```javascript
sha256(string|number|boolean|object)
sha256(string|number|boolean|object, 'base64')
sha256(string|number|boolean|object, 'base32')
sha256(string|number|boolean|object, 'hex')
```

Returns SHA-256 hash of input string/object in Base64 encoding \(default\), Base32 or Hex encoding. Non-string inputs are converted to strings. This function adds +1 to complexity count.

### Checksummed 160-bit hash

```javascript
$definition = ["sig", {
  "pubkey": "Ald9tkgiUZQQ1djpZgv2ez7xf1ZvYAsTLhudhvn0931w"
}];
$address = chash160($definition);

$aa_definition = ['autonomous agent', {
  ...
}];
$aa_address = chash160($aa_definition);
```

Returns a 160-bit checksummed hash of an object, it is most useful for calculating an address when you know its definition, e.g. if you programmatically define a new AA and want to know its address in order to immediately trigger it.

### Check trigger data existence

```javascript
exists(trigger.data.param)
```

Returns boolean `true` if the trigger data parameter with name `param` exists.

### is\_integer

```javascript
is_integer(number)
```

Returns boolean `true` if the number is without fractionals.

### is\_array

```javascript
is_array(object)
```

Returns boolean `true` if the object is an array.

### is\_assoc

```javascript
is_assoc(object)
```

Returns boolean `true` if the object is an associative array \(dictionary\).

### is\_valid\_address

```javascript
is_valid_address(string)
```

Returns boolean `true` if the string is valid Obyte wallet address.

### is\_aa

```javascript
is_aa(string)
```

Returns boolean `true` if the string is Autonomous Agent address.

### is\_valid\_amount

```javascript
is_valid_amount(number)
```

Returns boolean `true` if number is positive, integer, and below MAX\_CAP \(maximum cap that any token can have on Obyte platform\).

### is\_valid\_signed\_package

```javascript
is_valid_signed_package(signedPackage, address)
```

Returns `true` if `signedPackage` object is a valid signed package signed by address `address`, returns `false` otherwise \(the formula doesn't fail even if `signedPackage` doesn't have the correct format\). `address` must be a valid address, otherwise the expression fails with an error. This function adds +1 to complexity count.

`signedPackage` object is usually passed through the trigger and has the following structure:

```javascript
{
    "signed_message": {
        "field1": "value1",
        "field2": "value2",
        ...
    },
    "authors": [
        {
            "address": "2QHG44PZLJWD2H7C5ZIWH4NZZVB6QCC7",
            "authentifiers": {
                "r": "MFZ0eFJeLAgAmm6BJdvbEzNt7x0H2Fb5RQBBpMSmyVFMLM2r2SX5chU9hbEWXExkz/T2hXAk1qHmxkAbbpZw8w=="
            }
        }
    ],
    "last_ball_unit": "izgjyn9bpbJjwpKQV7my0Dq1VUHbzrLpWLrdR0fDydw=",
    "version": "2.0"
}
```

Here:

* `signed_message` is the message being signed, it can be an object, an array, or scalar;
* `authors` is an array of authors who signed the message \(usually one\), it has the same structure as unit authors and includes the signing address, authentifiers \(usually signatures\) and optionally definitions;
* `last_ball_unit`: optional unit of last ball that indicates the position on the DAG at which the message was signed. If definition is not included in `author`, it must be known at this point in the ledger history. If there is no `last_ball_unit` in `signedPackage`, including address definition as part of each `author` is required;
* `version`: always `2.0`.

Usually, `signedPackage` is created by calling `signMessage` function from `signed_message` module:

```javascript
var headlessWallet = require('headless-obyte');
var signed_message = require('ocore/signed_message.js');

signed_message.signMessage(message, address, headlessWallet.signer, true, function (err, signedPackage) {
    // handle result here
    trigger.data.signedPackage = signedPackage;
});
```

The function creates a correctly structured `signedPackage` object which can be added to `trigger.data`.

### is\_valid\_sig

```javascript
is_valid_sig(message, public_key, signature)
```

Returns `true` if `signature` is a correct ECDSA signature of `message` by the private key corresponding to `public_key`, returns `false` otherwise.

* `message` is a string corresponding to the message being signed, the function will hash the message with SHA-256 before verifying the signature. In case `message` is not a string, the formula will fail.
* `public_key` is a string containing the public key in a PEM format. For example:

  ```text
  -----BEGIN PUBLIC KEY-----
  MEowFAYHKoZIzj0CAQYJKyQDAwIIAQEEAzIABG7FrdP/Kqv8MZ4A097cEz0VuG1P\n\ebtdiWNfmIvnMC3quUpg3XQal7okD8HuqcuQCg==
  -----END PUBLIC KEY-----
  ```

  `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` can be omitted, spaces or carriage returns will be ignored. If `public_key` is not a string, is not for a supported curve, or doesn't have the required length then the formula will fail.

* `signature` is a string containing the signature in Base64 or hexadecimal format. In case signature is not a string or is not Base64 nor hexadecimal format, the formula will fail.

Supported algorithms:

ECDSA

`brainpoolP160r1, brainpoolP160t1, brainpoolP192r1, brainpoolP192t1, brainpoolP224r1, brainpoolP224t1, brainpoolP256r1, brainpoolP256t1, prime192v1, prime192v2, prime192v3, prime239v1, prime239v2, prime239v3, prime256v1, secp112r1, secp112r2, secp128r1, secp128r2, secp160k1, secp160r1, secp160r2, secp192k1, secp224k1, secp224r1, secp256k1, secp384r1, sect113r1, sect113r2, sect131r1, sect131r2, wap-wsg-idm-ecid-wtls1, wap-wsg-idm-ecid-wtls4, wap-wsg-idm-ecid-wtls6, wap-wsg-idm-ecid-wtls7, wap-wsg-idm-ecid-wtls8, wap-wsg-idm-ecid-wtls9`

RSA

`PKCS #1 - 512 to 4096 bits`

### vrf\_verify

```javascript
vrf_verify(seed, proof, pubkey)
```

Returns `true` if `proof` is valid, return false otherwise.

* `seed` is a string from which is derived a proof unique for this RSA key. The formula will fail in case `seed` is not a string or is empty.
* `proof` is an hexadecimal string value from 128 to 1024 characters \(depending of RSA key size\). Can be used with `number_from_seed` to obtain a verifiable random number.
* `pubkey` is a string containing the RSA public key in a PEM spki format. For example:

  ```text
  -----BEGIN PUBLIC KEY-----
  MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBANOJ1Y6/6uzcHDa7Q1gLO9z0KGOM51sO
  Pc2nfBF4RTobSVVpFnWtZF92r8iWCebwgSRSS9dEX6YIMIWNg11LbQ8CAwEAAQ==
  -----END PUBLIC KEY-----
  ```

  `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` can be omitted, spaces or carriage returns will be ignored. If `public_key` is not a string, is not for a supported curve, or doesn't have the required length then the formula will fail.

Supported algorithm: `RSAPKCS #1 - 512 to 4096 bits`

### is\_valid\_merkle\_proof

```javascript
is_valid_merkle_proof(element, proof);
```

Check if a given element is included in a merkle tree. The `proof` has the following structure `{root: string, siblings: array, index: number}` and would normally come from `trigger.data`.  One should check `is_valid_merkle_proof` and look up the merkle hash `root` from a data feed.

### bounce

```javascript
bounce(string);
```

Aborts the script's execution with error message passed as the function's argument. The received money will be bounced to sender \(less bounce fees\).

## Ocore functions

These functions are to be used in conjunction with `is_valid_sig` and `vrf_verify`.

### **signMessageWithEcPemPrivKey**

```javascript
const sig = require(‘ocore/signature.js’)
var signature = sig.signMessageWithEcPemPrivKey(message, encoding, pem_key)
```

Returns a signature that can be verified with is\_valid\_sig

`message` is a string to be signed.

`encoding`is either 'base64 or 'hex'

`pem_key` is a string containing ECDSA private key in PEM format

### **signMessageWithRsaPemPrivKey** 

```javascript
const sig = require(‘ocore/signature.js’)
var signature = sig.signMessageWithRsaPemPrivKey(message, encoding, pem_key)
```

Returns a signature that can be verified with is\_valid\_sig.

`message` is a string to be signed.

`encoding`is either 'base64 or 'hex'

`pem_key` is a string containing RSA private key in PEM format

### **vrfGenerate**

```javascript
const sig = require(‘ocore/signature.js’)
var proof = vrfGenerate(seed, pem_key)
```

_\*\*_Returns a proof that can verified with vrf\_verify.

`seed` is a string from which is derived the unique proof

`pem_key` _\*\*_is a string containing RSA private key in PEM format 

**Key generation with openssl**

Keys for the functions above can be generated with openssl.

RSA \(mandatory for vrfGenerate and vrf\_verify\):

private key

`openssl genrsa -out priv_key.pem 2048`

Replace 2048 by any key length from 512 to 4096.

public key

`openssl rsa -in priv_key.pem -outform PEM -pubout -out pub_key.pem`

ECDSA:

private key

openssl ecparam -name prime256v1 -genkey -noout -out priv\_key.pem  
 Replace prime256v1 by any curve identifier listed above

public key

`openssl ec -in priv_key.pem -pubout -out pub_key.pem`

## Conversions

### To-string conversion

Some functions and operators that expect strings as inputs need to convert non-string inputs to strings:

* numbers are converted to strings using their decimal representation.  For numbers whose exponent is greater than or equal to 21 or less than or equal to -7, exponential representation is used.
* booleans are converted to strings `true` and `false`.
* objects become strings `true`.

### To-number conversion

Some functions and operators that expect numbers as inputs need to convert non-number inputs to numbers:

* booleans `true` and `false` are converted to numbers `1` and `0` respectively.
* objects become 1.
* strings become numbers,  `+` can be used to force the conversion, e.g. `+'3'` becomes `3`, `+false` becomes `0`.

### To-boolean conversion

0 and empty string become `false`, all other values become `true`. Any value that would convert to `true` is called truthy, otherwise falsy.

## Comments

Line comments

```javascript
// this is a comment line
$x = 1; // this part of line is a comment
```

as well as block comments are supported:

```javascript
/*
A comment block
*/
```

## References to external variables

### trigger.address

The address of the sender who sent money to this AA. If the sending unit was signed by several addresses, the first one is used.

### trigger.initial\_address

The address of the sender who sent money to the initial AA of a chain of AAs. Same as `trigger.address` if there was no chain. When an AA sends money to another AA, `trigger.initial_address` remains unchanged.

### trigger.unit

The unit that sent money to this AA.

### trigger.output

```javascript
trigger.output[[asset=assetID]].field
trigger.output[[asset!=assetID]].field
```

Output sent to the AA address in the specified asset.

`assetID` can be `base` for bytes or any expression that evaluates to asset ID.

`field` can be `amount` or `asset` or omitted. If omitted, `amount` is assumed. If the trigger unit had several outputs in the same asset to this AA address, their amounts are summed.

The search criteria can be `=` \(`asset=assetID`\) or `!=` \(`asset!=assetID`\).

Examples:

```javascript
trigger.output[[asset=base]]
trigger.output[[asset=base]].amount
trigger.output[[asset='j52n7Bfec9jW']]
trigger.output[[asset=$asset]]
trigger.output[[asset!=base]]
trigger.output[[asset!=base]].amount
if (trigger.output[[asset!=base]].asset == 'ambiguous'){
    ...
}
```

If there is no output that satisfies the search criteria, the returned `.amount` is 0 and the returned `.asset` is a string `none`. Your code should check for this string if necessary.

If there is more than one output that satisfies the search criteria \(which is possible only for `!=`\), the returned `.asset` is a string `ambiguous`. Your code should check for this string if necessary. Trying to access `.amount` of an ambiguous asset fails the script.

### trigger.data

Data sent with the trigger unit in its `data` message. `trigger.data` returns the entire data object, `trigger.data.field1.field2` or `trigger.data.field1[expr2]` tries to access a deeper nested field:

* if it is an object, object is returned;
* if it is a scalar \(string, number, or boolean\), scalar is returned;
* if it doesn't exist, `false` is returned.
* `exists` function can be used to check if param [exists](oscript-language-reference.md#check-trigger-data-existence).

For example, if the trigger unit had this data message

```javascript
{
    "app": "data",
    "payload": {
        "field1": {
            "field2": "value2",
            "abc": 88
        },
        "abc": "def"
    },
    "payload_hash": "..."
}
```

`trigger.data` would be equal to

```javascript
{
    "field1": {
        "field2": "value2",
        "abc": 88
    },
    "abc": "def"
}
```

`trigger.data.field1` would be equal to

```javascript
{
    "field2": "value2",
    "abc": 88
}
```

`trigger.data.field1.field2` would be equal to string `value2`,

`trigger.data.field1['a' || 'bc']` would be equal to number `88`,

`trigger.data.field1.nonexistent` would be equal to boolean `false`,

`trigger.data.nonexistent.anotherfield` would be equal to boolean `false`.

### mci

MCI of the trigger unit, which is the same as MCI of MC unit the response unit \(if any\) will be attached to.

### timestamp

Timestamp of the MC unit that recently became stable, this is the unit whose stabilization triggered the execution of this AA. This is the same unit the response unit \(if any\) will be attached to. It's number of seconds since Epoch - Jan 01 1970. \(UTC\).

### mc\_unit

Hash of the MC unit that includes \(or is equal to\) the trigger unit.

### storage\_size

The size of AA’s storage occupied before the current invocation

### number\_of\_responses

Built-in variable says how many responses were already generated in response to a primary trigger and might help to avoid exceeding the limit of 10 responses per primary trigger.

### this\_address

The address of this AA.

### response\_unit

The hash of the unit that will be generated by the AA in response to the trigger. This variable is available only in state script. Any references to this variable in any other scripts will fire an error.

### definition

```javascript
definition[trigger.address]
```

It allows to inspect the definition of any address using `definition['ADDRESS']` syntax.

Examples:

```javascript
definition[trigger.address][0] == 'autonomous agent'
definition[trigger.address][1].base_aa == 'EXPECTED_BASE_AA'.
```

### asset

```javascript
asset[expr].field
asset[expr][field_expr]
```

Extracts information about an asset. This adds +1 to complexity. `expr` is `base` for bytes or an expression that evaluates to an asset ID.

`field` is on of the following, `field_expr` should evaluate to one of the following:

* `exists`: boolean, returns `false` if asset ID is invalid;
* `cap`: number, total supply of the asset.  For uncapped assets, 0 is returned;
* `is_private`: boolean, is the asset private?
* `is_transferrable`: boolean, is the asset transferrable?
* `auto_destroy`: boolean, does the asset gets autodestroyed when sent to definer address?
* `fixed_denominations`: boolean, is the asset issued in fixed denominations? Currently AAs can't send fixed denomination assets, but if `issued_by_definer_only` is `false` then somebody else can issue them.
* `issued_by_definer_only`: boolean, is the asset issued by definer only?
* `cosigned_by_definer`: boolean, should each transfer be cosigned by definer?
* `spender_attested`: boolean, should each holder be attested?
* `is_issued`: boolean, is any amount of the asset already issued?
* `definer_address`: string, returns wallet address of the definer.

Examples:

```javascript
asset[base].cap
asset['base'].cap
asset['abc'].exists
asset['n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY='].is_issued
asset['n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY=']['is_' || 'issued']
asset['n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY=']['is_' || 'private']
```

If the asset ID is valid, but does not exist then `false` is returned for any field.

### data\_feed

```javascript
data_feed[[oracles=listOfOracles, feed_name=nameOfDataFeed, ...]]
```

Finds data feed value by search criteria. This adds +1 to complexity.

There are multiple search criteria listed between the double brackets, their order is insignificant.

* `oracles`: string, list of oracle addresses delimited by `:` \(usually only one oracle\). `this_address` refers to the current AA;
* `feed_name`: string, the name of the data feed;
* `feed_value`: string or number, optional, search only for this specific value of the data feed;
* `min_mci`: number, optional, search only since the specified MCI;
* `ifseveral`: string, optional, `last` or `abort`, what to do if several values found that match all the search criteria, return the last one or abort the script with error, default is `last`
* `ifnone`: string or number or boolean, optional, the value to return if nothing is found.  By default, this results in an error and aborts the script;
* `what`: string, optional, `value` or `unit`, what to return, the data feed value or the unit where it was posted, default is `value`;
* `type`: string, optional, `auto` or `string`, what type to return, default is `auto`.  For `auto`, data feed values that look like valid IEEE754 numbers are returned as numbers, otherwise they are returned as strings.  If `string`, the returned value is always a string.  This setting affects only the values extracted from the database; if `ifnone` is used, the original type of `ifnone` value is always preserved.

Data feeds are searched before the MCI of the triggering unit \(inclusively\). If there are several AAs stemming from the same MCI, previous AA responses are also searched.

Examples:

```javascript
data_feed[[oracles='JPQKPRI5FMTQRJF4ZZMYZYDQVRD55OTC', feed_name='BTC_USD']]
data_feed[[oracles=this_address, feed_name='score']]
data_feed[[oracles='JPQKPRI5FMTQRJF4ZZMYZYDQVRD55OTC:I2ADHGP4HL6J37NQAD73J7E5SKFIXJOT', feed_name='timestamp']]
```

### in\_data\_feed

```javascript
in_data_feed[[oracles=listOfOracles, feed_name=nameOfDataFeed, feed_value>feedValue, ...]]
```

Determines if a data feed can be found by search criteria. Returns `true` or `false`. This adds +1 to complexity.

There are multiple search criteria listed between the double brackets, their order is insignificant.

* `oracles`: string, list of oracle addresses delimited by `:` \(usually only one oracle\). `this_address` refers to the current AA;
* `feed_name`: string, the name of the data feed;
* `feed_value`: string or number, search only for values of the data feed that are `=`, `!=`, `>`, `>=`, `<`, or `<=` than the specified value;
* `min_mci`: number, optional, search only since the specified MCI.

Data feeds are searched before the MCI of the triggering unit \(inclusively\). If there are several AAs stemming from the same MCI, previous AA responses are also searched.

Examples:

```javascript
in_data_feed[[oracles='JPQKPRI5FMTQRJF4ZZMYZYDQVRD55OTC', feed_name='BTC_USD', feed_value>12345.67]]
in_data_feed[[oracles=this_address, feed_name='score', feed_value=$score]]
in_data_feed[[oracles='JPQKPRI5FMTQRJF4ZZMYZYDQVRD55OTC:I2ADHGP4HL6J37NQAD73J7E5SKFIXJOT', feed_name='timestamp', feed_value>=1.5e9]]
```

### attestation

```javascript
attestation[[attestors=listOfAttestors, address=attestedAddress, ...]].field
attestation[[attestors=listOfAttestors, address=attestedAddress, ...]][field_expr]
```

Finds an attestation by search criteria. This adds +1 to complexity.

There are multiple search criteria listed between the double brackets, their order is insignificant.

* `attestors`: string, list of attestor addresses delimited by `:` \(usually only one attestor\). `this_address` refers to the current AA;
* `address`: string, the address that was attested;
* `ifseveral`: string, optional, `last` or `abort`, what to do if several matching attestations are found, return the last one or abort the script with error, default is `last`
* `ifnone`: string or number or boolean, optional, the value to return if nothing is found.  By default, this results in an error and aborts the script;
* `type`: string, optional, `auto` or `string`, what type to return, default is `auto`.  For `auto`, attested field values that look like valid IEEE754 numbers are returned as numbers, otherwise they are returned as strings.  If `string`, the returned value is always a string.  This setting affects only the values extracted from the database; if `ifnone` is used, the original type of `ifnone` value is always preserved.

`field` string or `field_expr` expression are optional and they indicate the attested field whose value should be returned. Without `field` or `field_expr`, `true` is returned if an attestation is found.

If no matching attestation is found, `ifnone` value is returned \(independently of `field`\). If there is no `ifnone`, `false` is returned.

If a matching attestation exists but the requested field does not, the result is as if the attestation did not exist.

Attestations are searched before the MCI of the triggering unit \(inclusively\). If there are several AAs stemming from the same MCI, previous AA responses are also searched.

Examples:

```javascript
attestation[[attestors='UOYYSPEE7UUW3KJAB5F4Y4AWMYMDDB4Y', address='BI2MNEVU4EFWL4WSBILFK7GGMVNS2Q3Q']].email
attestation[[attestors=this_address, address=trigger.address]]
attestation[[attestors='JEDZYC2HMGDBIDQKG3XSTXUSHMCBK725', address='TSXOWBIK2HEBVWYTFE6AH3UEAVUR2FIF', ifnone='anonymous']].steem_username
attestation[[attestors='JEDZYC2HMGDBIDQKG3XSTXUSHMCBK725', address='TSXOWBIK2HEBVWYTFE6AH3UEAVUR2FIF']].reputation
```

### balance

```javascript
balance[asset]
balance[aa_address][asset]
```

Returns the balance of an AA in the specified asset. If `aa_address` is omitted, the current AA is assumed. `asset` can be `base` for bytes, asset id for any other asset, or any expression that evaluates to an asset id or `base` string.

This adds +1 to complexity count.

The returned balance includes the outputs received from the current trigger.

Examples:

```javascript
balance[base]
balance['n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY=']
balance['JVUJQ7OPBJ7ZLZ57TTNFJIC3EW7AE2RY'][base]
```

### input, output

```javascript
output[[asset=assetID, amount>minAmount, address=outputAddress]].field
input[[asset=assetID, amount=amountValue, address=inputAddress]].field
```

Tries to find an input or output in the current unit by search criteria.

These language constructs are available only in non-AA formulas in smart contracts \(`["formula", ...]` clause\).

There are multiple search criteria listed between the double brackets, their order is insignificant. All search criteria are optional but at least one must be present.

* `asset`: string, asset of input or output, can be `base` for bytes.  Comparison operators can be only `=` or `!=`;
* `address`: string, the address receives an output or spends an input, can be `this_address`.  Comparison operators can be only `=` or `!=` \(other addresses\);
* `amount`: number, the condition for the amount of an input or output.  Allowed comparison operators are: `=`, `!=`, `>`, `>=`, `<`, `<=`.

`field` is one of `amount`, `address`, and `asset`. It indicates which information about the input or output we are interested in.

If no input/output is found by search criteria or there is more than one matching entry, the formula fails.

Examples:

```javascript
input[[asset=base]].amount
output[[asset = base, address=GFK3RDAPQLLNCMQEVGGD2KCPZTLSG3HN]].amount
output[[asset = base, address='GFK3RDAPQLLNCMQEVGGD2KCPZTLSG3HN']].amount
output[[asset = 'n9y3VomFeWFeZZ2PcSEcmyBb/bI7kzZduBJigNetnkY=', amount>=100]].address
```

## Limits

* strings cannot be longer than 4096 characters;
* state var value strings cannot be longer than 1024 characters;
* state var names cannot be longer than 128 characters;
* numbers must be in IEEE754 double range;
* all intermediary calculations are rounded to 15 significant digits;
* total complexity of all scripts cannot exceed 100;
* total number of operations of all scripts cannot exceed 2000.

Any attempt to exceed these limits will result in script's failure.

## Deployment

AA code can be deployed with [Oscript](https://oscript.org)[ \(mainnet\)](https://oscript.org)[ editor](https://oscript.org) and [Oscript](https://testnet.oscript.org/)[ \(testnet\)](https://testnet.oscript.org/)[ editor](https://testnet.oscript.org/), but AA code can also be deployed by sending a unit that includes a message with `app=definition`. This can be done with headless wallet or with AA itself, which has the definition of new AA in its payload.

```javascript
const headlessWallet = require('headless-obyte');
const eventBus = require('ocore/event_bus.js');
const objectHash = require('ocore/object_hash.js');

function postAA() {
    let definition = ["autonomous agent", {
        bounce_fees: { base: 10000 },
        messages: [
            ...
        ]
    }];
    let payload = {
        address: objectHash.getChash160(definition),
        definition: definition
    };
    let objMessage = {
        app: 'definition',
        payload_location: 'inline',
        payload_hash: objectHash.getBase64Hash(payload),
        payload: payload
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

eventBus.on('headless_wallet_ready', postAA);
```

There is no error if the same definition is posted twice.

The user who deployed the AA doesn't have any privileges over the AA aside from those directly coded in the AA source code.

Once deployed, the AA definition can never be changed.

Any outputs sent to the AA address before its definition was revealed \("before" means before the MCI of the first unit where the definition was revealed\) are treated like regular outputs, they just increase the address'es balance but do not trigger any action, even delayed action after the definition was revealed. This might be a convenient way to fill the AA with some coins before actually launching it.

It is possible for AA to deploy another AA, which is especially useful when creating [parameterized AAs](oscript-language-reference.md#base_aa) for previously deployed AA templates:

```javascript
["autonomous agent", {
    bounce_fees: { base: 10000 },
    messages: [
        {
            app: "definition",
            payload: {
                definition: ["autonomous agent", {
                    base_aa: "BASE_AA_TEMPLATE_ADDRESS",
                    params: {
                        name1: "{trigger.data.name1}",
                        name2: "{trigger.data.name2}",
                        ...
                    }
                }]
            }
        },
        ...
    ]
}]
```

In case you wish to enter expressions that doesn't get evaluated with the deployment of new AA, but as actual expressions for a new AA, instead of `"{expression}"` write an expression that outputs another expression `"{'{expression}'}"`.

