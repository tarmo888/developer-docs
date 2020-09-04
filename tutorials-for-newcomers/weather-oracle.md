# Weather oracle

Hi, everybody. Today we will create our own oracle. To do this, we need a bot-example. Let’s install it.

```bash
git clone https://github.com/byteball/bot-example
cd bot-example
npm install
cp .env.testnet .env
```

Our oracle will accept a message like City: timestamp, where timestamp is the time to publish.  
Let’s write a message handler

```javascript
headlessWallet.readSingleAddress(address => {
  my_address = address;
  console.error('my_address', address)
});

eventBus.on('text', (from_address, text) => {
  text = text.trim();
  const device = require('ocore/device.js');
  const args = text.toLowerCase().split(':');
  if (args.length === 2){
     switch (args[0]) {
        case 'berlin':
        case 'moscow':
        case 'helsinki':
        case 'washington':
           if(parseInt(args[1]) <= Date.now()){
              console.error('dateNow', Date.now());
              device.sendMessageToDevice(from_address, 'text', "Incorrect time");
           }else {
              arrQueue.push({city: args[0], time: args[1], device_address: from_address});
              device.sendMessageToDevice(from_address, 'text', "ok");
           }
           break;
        default:
           device.sendMessageToDevice(from_address, 'text', "City not support");
           break;
     }
  }else {
     device.sendMessageToDevice(from_address, 'text', "Incorrect command");
  }
});
```

Add a variable to store the queue

```javascript
let arrQueue = [];
```

Now add to the queue

```javascript
if(parseInt(args[1]) <= Date.now()){
  device.sendMessageToDevice(from_address, 'text', "Incorrect time");
}else {
  arrQueue.push({city: args[0], time: args[1], device_address: from_address});
  device.sendMessageToDevice(from_address, 'text', "ok");
}
```

Now we need to check the queue once a minute

```javascript
function checkQueue(){
  arrQueue.forEach((el, index) => {
     if(el.time <= Date.now()){
        let result = JSON.parse(body);
        postDataFeed(el.city, el.time, result.current.temp_c);
     }
  });
}
setTimeout(checkQueue, 60000);
```

Now we come to the main point. We need to publish data\_feed

```javascript
function postDataFeed(city, time, temp){
  const network = require('ocore/network.js');
  const composer = require('ocore/composer.js');

  let data_feed = {};
  data_feed[city + '_' + time] = temp;
  let params = {
     paying_addresses: [my_address],
     outputs: [{address: my_address, amount: 0}],
     signer: headlessWallet.signer,
     callbacks: composer.getSavingCallbacks({
        ifNotEnoughFunds: console.error,
        ifError: console.error,
        ifOk: function(objJoint){
           network.broadcastJoint(objJoint);
        }
     })
  };
  let objMessage = {
     app: "data_feed",
     payload_location: "inline",
     payload: data_feed
  };
  params.messages = [objMessage];
  composer.composeJoint(params);
}
```

What is happening?

1\) We create an object in data\_feed and keep the temperature in it.

```javascript
let data_feed = {};
data_feed[city + '_' + time] = temp;
```

2\) We send the payment to ourselves as we only need to publish data\_feed

```javascript
paying_addresses: [my_address],
outputs: [{address: my_address, amount: 0}]
```

3\) Create a message with data and signature

```javascript
let objMessage = {
     app: "data_feed",
     payload_location: "inline",
     payload: data_feed
  };
```

4\) Publish

```javascript
composer.composeJoint(params);
```

That’s all. Your first oracle is ready. For the experiment, you can add more cities or make paid access. All code is available on [github](https://github.com/xJeneKx/tutorial-3).Did you like the lesson? Any questions? What topics have caused you difficulties and it is worth talking about them? Write in the comments and I will help you. Also join my telegram chat - [@obytedev](https://t.me/obytedev).

