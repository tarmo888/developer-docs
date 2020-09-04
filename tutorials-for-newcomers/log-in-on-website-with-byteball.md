# Logging into website

Hey guys! Today we will learn how to log in using a Obyte wallet, and also make paid access to the article. To do this, we need project that i have prepared for you [Tutorial-2-source](https://github.com/xJeneKx/Tutorial-2-source). Install it.

```bash
git clone https://github.com/xJeneKx/Tutorial-2-source
cd Tutorial-2-source
npm install
```

If you run project you will see a blog with articles. After opening any article you will see a request to log in. Let’s create an authorization. First we need a pairing code, you could see it on startup

![](https://lh4.googleusercontent.com/UB0mdQRYTSrDVgC2UTEFQub1-vGYV25q4fcnZnWivCVwn7-Gh8QNTgMSDtRTkgPEO5y5lF9KD_W19WgqMGCg5Phd2i9JVjSSLHI3JHz5Qg-QqnyqFjJNzI_P3p_2bU4IKzPNoJBZ)

Copy it and change our code a bit

```javascript
pairingCode = 'Aly99HdWoCFaxJiIHs1ldREAN/sMDhGsRHNQ2RYU9gCj@byteball.org/bb#' + code;
```

```javascript
eventBus.on('paired', (from_address, pairing_secret) => {
  console.error('paired', from_address, pairing_secret);
});
```

Run and see the link. Сlick on it and bot will be added. Now let’s look at the console.

![](https://lh3.googleusercontent.com/CoHNgrayZGvQFYDFzwREmddk4ghl3BTq4AiJFjS0ApNqltnBTuqJTPWiZJEs6wG6OCFg8M2IKwOWoypRgYkXktt2A5j_IsIJLpEsa7uW5PjZgjUG2DfQNVtikCwnb1mhruSMNpeT)

Add the link

```javascript
b = '<br>Please login using this pairing code: <a href="obyte:' + pairingCode + '">' + pairingCode + '</a>';
```

And QR

```javascript
let dataURL = await QRCode.toDataURL("obyte:" + pairingCode);
b = '<br>Please login using this pairing code: <a href="obyte:' + pairingCode + '">' + pairingCode + '</a><br><img src="' + dataURL + '">';
```

Save our device address, we will need it later.

```javascript
eventBus.on('paired', (from_address, pairing_secret) => {
  let device = require('ocore/device');
  assocCodeToDeviceAddress[pairing_secret] = from_address;
  device.sendMessageToDevice(from_address, 'text', 'ok');
});
```

Now we need to tell the browser that we are logged in  
To implement this, we add function in start.js

```javascript
async function amILogged(ctx) {
  let url = ctx.request.url;
  let match = url.match(/code=([0-9]+)/);
  if (match && assocCodeToDeviceAddress[match[1]]) {
     ctx.cookies.set('ac', match[1]);
     ctx.body = 'true';
  } else {
     ctx.body = 'false';
  }
}
```

And also one more routing

```javascript
app.use(mount('/amilogged', amILogged));
```

In article.html we will add a request to check

```javascript
var step = '<%= step %>';
var code = '<%= code %>';
if (step === 'login') {
  setInterval(() => {
     fetch('/amilogged?code=' + code)
        .then((response) => {
           return response.text();
        }).then(result => {
        if (result === 'true') {
           location.reload(true);
        }
     })
        .catch(alert);
  }, 5000);
}
```

What’s going on here? The user scans the QR code, adds our bot. Bot associates code with his device\_address. In the browser, we knock and ask: did we logged in? and adding cookies.

Now we need to create an address and associate it with the user and the article.  
Add a variable

```javascript
let assocCodeAndNumberToAddress = {};
```

Add the address and display it

```javascript
step = 'paid';
let address = await getAssocAddress(assocCode, 1);
let dataURL = await QRCode.toDataURL("obyte:" + address + '?amount=100');
b = '<br>Please pay for the article. <br>Address: ' + address + '<br>Amount: 100<br><img src="' + dataURL + '"><br>' +
  '<a href="obyte:' + address + '?amount=100">Pay</a>';
```

And finally the function getAssocAddress

```javascript
function getAssocAddress(assocCode, number) {
  return new Promise(resolve => {
     let name = assocCode + '_' + number;
     if (assocCodeAndNumberToAddress[name]) {
        return resolve(assocCodeAndNumberToAddress[name]);
     } else {
        headlessWallet.issueNextMainAddress(address => {
           assocCodeAndNumberToAddress[name] = address;
           return resolve(address);
        });
     }
  });
}
```

This time we will not wait for the stabilization of the unit and immediately open access.  
Add a variable

```javascript
let assocCodeToPaid = {};
```

Add a check that the user has paid for

```javascript
async function amIPaid(ctx) {
  let code = ctx.cookies.get('ac');
  ctx.body = (code && itPaid(code)) ? 'true' : 'false';
}

function itPaid(code) {
  return !!assocCodeToPaid[code];
}
```

Add a routing

```javascript
app.use(mount('/amipaid', amIPaid));
```

And payment processing

```javascript
eventBus.on('new_my_transactions', (arrUnits) => {
  const device = require('ocore/device.js');
  db.query("SELECT address, amount, asset FROM outputs WHERE unit IN (?)", [arrUnits], rows => {
     rows.forEach(row => {
        if (row.amount === 100 && row.asset === null) {
           for (let key in assocCodeAndNumberToAddress) {
              if (assocCodeAndNumberToAddress[key] === row.address) {
                 let assocCode = key.split('_')[0];
                 assocCodeToPaid[assocCode] = true;
                 device.sendMessageToDevice(assocCodeToDeviceAddress[assocCode], 'text', 'I received your payment');
                 return;
              }
           }
        }
     })
  });
});
```

That’s all, make a working second article - will be your homework:\) You should get this:  
[https://www.youtube.com/watch?v=vvB6A1wBecQ](https://www.youtube.com/watch?v=vvB6A1wBecQ)

All the code you can find on [github](https://github.com/xJeneKx/Tutorial-2-full). Did you like the lesson? Any questions? What topics have caused you difficulties and it is worth talking about them? Write in the comments and I will help you. Also join my telegram chat - [@obytedev](https://t.me/obytedev).

