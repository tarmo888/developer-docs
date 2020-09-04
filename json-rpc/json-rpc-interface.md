---
description: >-
  If you are developing a service on Obyte and your programming language is
  node.js, your best option is to just require() the ocore modules that you
  need.
---

# Exposing RPC interface

For exchanges and other custody wallets, there is a specialized [JSON RPC service](running-rpc-service.md). However it is limited and exposes only those functions that the exchanges need.

If you are developing a service on Obyte and your programming language is node.js, your best option is to just `require()` the ocore modules that you need \(most likely you need [headless-obyte](https://github.com/byteball/headless-byteball) and various modules inside [ocore](https://github.com/byteball/byteballcore)\). This way, you'll also be running a Obyte node in-process.

If you are programming in another language, or you'd like to run your Obyte node in a separate process, you can still access many of the functions of `headless-obyte` and `ocore`by creating a thin RPC wrapper around the required functions and then calling them via JSON-RPC.

## Get started

To get started, we can add RPCify to existing project or directly to headless-obyte with this command:

```bash
npm install https://github.com/byteball/rpcify.git
```

See the documentation about [RPCify](https://github.com/byteball/rpcify) for more details.

## Exposing functions and events

To expose the required functions via JSON-RPC, create a project that has `headless-obyte`, `ocore` \(and any other ocore modules\) and [RPCify](https://github.com/byteball/rpcify) as dependencies:

```javascript
var rpcify = require('rpcify');
var eventBus = require('ocore/event_bus.js');

// this is a module whose methods you want to expose via RPC
var headlessWallet = require('headless-obyte'); // when headless-obyte is dependency of your project
//var headlessWallet = require('../start.js'); // when this script is in headless-obyte tools folder
var balances = require('ocore/balances.js'); // another such module

// most of these functions become available only after the passphrase is entered
eventBus.once('headless_wallet_ready', function(){
	// start listening on RPC port
	rpcify.listen(6333, '127.0.0.1');

	// expose some functions via RPC
	rpcify.expose(headlessWallet.issueChangeAddressAndSendPayment);
	rpcify.expose(balances.readBalance, true);
	rpcify.expose(balances.readAllUnspentOutputs);
	rpcify.expose([
		headlessWallet.readFirstAddress,
		headlessWallet.readSingleWallet,
		headlessWallet.issueOrSelectAddressByIndex
	], true);

	// expose some events 
	rpcify.exposeEvent(eventBus, "my_transactions_became_stable");
	rpcify.exposeEvent(eventBus, "new_my_transactions");
});
```

## Calling with HTTP requests

From another Node.js app, calling the function would look something like this:

```javascript
var rpc    = require('json-rpc2');
var client = rpc.Client.$create(6333, '127.0.0.1');

client.call('issueChangeAddressAndSendPayment', [asset, amount, to_address, device_address], function(err, unit) {
    ...
});
```

## Listening via WebSockets

From another Node.js app, sending function calls and listening to events would look something like this:

```javascript
var WebSocket = require('ws');
var ws        = new WebSocket("ws://127.0.0.1:6333");

ws.on('open', function onWsOpen() {
	console.log("ws open");
	ws.send(JSON.stringify({
		"jsonrpc":"2.0",
		"id":1,
		"method":"readBalance",
		"params":["LUTKZPUKQJDQMUUZAH4ULED6FSCA2FLI"]
	})); // send a command
});

ws.on('error', function onWsError(e){
	console.log("websocket error"+e);
})

ws.on('message', function onWsMessage(message){ // JSON responses
	console.error(message);
	// {"jsonrpc":"2.0","result":{"base":{"stable":0,"pending":0}},"error":null,"id":1} // response to readBalance
  // {"event":"my_transactions_became_stable","data":[["1pLZa3aVicNLE6vcClG2IvBe+tO0V7kDsxuzQCGlGuQ="]]} // event my_transactions_became_stable
});
```

