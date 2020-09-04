---
description: >-
  Obyte can accept Byte payments for you and notify you when the Bytes have
  arrived to your account. There is also an easy-to-setup Wordpress/Woocommerce
  plugin.
---

# Payments gateway

## Requirements

* A Obyte light wallet to provide us with your merchant Obyte address \(you do not need to run the heavy full wallet, we are running it for you\). Download the wallet from [https://obyte.org](https://obyte.org).
* If you want to use our Woocommerce plugin skip reading now and go to: [https://wordpress.org/plugins/woobytes-gateway/](https://wordpress.org/plugins/woobytes-gateway/)
* Upon going live a _merchant\_id_ and a _secret\_key_. Contact us on [https://obyte.org/discord](https://obyte.org/discord) \#merchants channel to get your `merchant_id` and you `secret_key`.

## Integration process

**Step 1:** Either in _test_ mode or in _live_ mode, customize the Obyte quick payment button script with your merchant parameters and insert it in your checkout web page. Obyte quick payment button init script has to be inserted in the header section of your checkout page and the button itself can be displayed anywhere on the page in a Div container. The _test_ mode triggers the whole notification process except there is **no real payment**. Once you are done with your tests, switch to _live_ mode by changing the "mode" parameter from "_test_" to "_live_" in the button init script.  
See example.html: [https://github.com/tarmo888/bbfm/tree/master/bbfm\_www/example](https://github.com/tarmo888/bbfm/tree/master/bbfm_www/example)

**Step 2:** We notify you \(upon your choice\), either by mail and/or by a GET or a POST to your server, when a payment for your sale has arrived, we simultaneously send you the payment to your merchant address, minus our fee. At this stage you receive a first notification with an _unconfirmed_ "result" field. When the payment is confirmed by the network you receive a second notification with a _ok_ "result" field. Such notification of a confirmed payment typically takes between 5 to 15 minutes from the customer click to pay \(live mode\) or 1 minute \(in test mode\). You then have to handle the notification on your end so that you actualy process your customer order. You can process the delivery as soon as you received the _ok_ notification \(processing on _unconfirmed_ notification is at your own risk!\). In any case always check on your end that _**received\_amount**_ does match _**amount\_asked\_in\_B**_ before processing the order or contact your customer for missing Bytes \(the event of an incomplete payment is not an error case on our end but it must be one on your end\).

## Parameters

### Step 1: Obyte payment button field list

* **order\_UID** \(mandatory\): Your merchant order ID _\(should match /^\[a-zA-Z0-9-\]{4,20}$/\)_.
* **merchant\_return\_url** \(optional\): The full URL we will POST or GET to notify you a payment arrived.
* **mode\_notif** \(optional, default is get\): Shall we GET or POST the notification to your server? _\(should match get \| post\)_.
* **currency** \(optional, default B\): Currency from which convert the amount into Bytes _\(should be B \(for Bytes\), BTC, ETH, USD, EUR, PLN, KRW, GBP, CAD, JPY, RUB, TRY, NZD, AUD, CHF, UAH, HKD, SGD, NGN, PHP, MXN, BRL, THB, CLP, CNY, CZK, DKK, HUF, IDR, ILS, INR, MYR, NOK, PKR, SEK, TWD, ZAR, VND, BOB, COP, PEN, ARS, ISK\)_. Conversion is done automaticaly in real time at rates from [CoinPaprika](https://api.coinpaprika.com/).
* **merchant\_email** \(optional\): Fill in your email address if you want to be notified by email when payment arrives OR to get an alert email if we were not able to GET or POST to notify you on your _**merchant\_return\_url**_. Take care to whitelist emails from "noreply@obyte-for-merchants.com" \(you can usually do that by adding this email to your contacts\).
* **amount** \(mandatory\): The amount charged to your customer in _**currency** \(should match /^\[0-9\]+.?\[0-9\]\*$/\)_. Minimum amount is 1500 Bytes \(or its equivalent in _currency_\) Maximum amount is 1000 GBytes.
* **byteball\_merchant\_address** \(mandatory\): Your unique Obyte merchant address to receive your payment.
* **merchant\_id** \(mandatory\): In _test_ mode _**merchant\_id**_ is _test\_merchant_. Contact us on [https://obyte.org/discord](https://obyte.org/discord) \#merchants channel to get a merchant id when you first go live.
* **mode** \(mandatory\): live \| test. Are you testing the implementation or are you running the Obyte payment button from a production server? _\(should match live or test and will be returned **as is** on **merchant\_return\_url**\)_.
* **attach** \(optional\): free text field \(255 chars\). Will be returned as is for your own convenience \(must match /^\[a-zA-Z0-9\_.@,=?& -\]{0,255}$/\).
* **qrcode** \(optional\): you can here indicate your own qrcode url mask to use the qrcode api of your choice, using {text} as url placeholder. By default, the Google QRCode API is used, that is to say the value is: '[https://chart.googleapis.com/chart?chs=200x200&cht=qr&chl={text}&chld=H](https://chart.googleapis.com/chart?chs=200x200&cht=qr&chl={text}&chld=H)' But you can change it and choose for instance the Chinese liantu API by setting the value: '[http://qr.liantu.com/api.php?text={text}&w=100](http://qr.liantu.com/api.php?text={text}&w=100)'

### Step 2: Returned field list on _**merchant\_return\_url**_

* **mode**: live \| test   

  _**WARNING! WARNING! WARNING!**_ Always check this field as in test mode the notification of payment received is always sent for integration and test purpose without any effective payment. Do not use the test mode on a production merchant server.

* **result**: nok \| incoming \| unconfirmed \| ok   

  _'nok'_: check **error\_msg** for text error.  

  _'incoming'_: the payment has been sent to us by your customer but it is still waiting for network confirmation.   

  _'unconfirmed'_: the payment has been sent to you but it is still waiting for network confirmation.  

  _'ok'_: final step. The payment unit we sent to you has been confirmed by the network. Process the order on your end.  

* **error\_msg** \(occurence: 1 on _result=nok_ or 0\): human readable text field
* **order\_UID**: Your merchant order ID
* **currency**: B \(for Bytes, see rest above\).
* **amount\_asked\_in\_currency** \(occurence: 0 on _currency=B_ or 1\): decimal  

  _**WARNING! WARNING! WARNING!**_ Always check _**amount\_asked\_in\_currency**_ does match the amount of the order in your own database and in your expected currency.

* **currency\_B\_rate** \(occurence: 0 on _currency=B_ or 1\): decimal
* **amount\_asked\_in\_B**: integer \(in Bytes\)
* **fee**: our fee \(in Bytes\)
* **received\_amount**: the amount we received \(in Bytes\).  

  _**WARNING! WARNING! WARNING!**_ check on your end that _**received\_amount**_ does match _**amount\_asked\_in\_B**_ before processing the order. Contact your customer if some Bytes are missing from what you expected \(the event of an incomplete payment is not an error case on our end but it must be one on your end\).

* **receive\_unit**: the DAG unit that shows the payment from your customer to our Obyte address.
* **amount\_sent**: the amount sent to your _**byteball\_merchant\_address**_ \(_**received\_amount**_ minus _**fee**_\)
* **unit**: the DAG unit that shows the payment from our Obyte address to your _**byteball\_merchant\_address**_
* **sha256\_digest**: the digest of the notification is computed as  sha256\(secret\_key+order\_UID+merchant\_return\_url+merchant\_email\) where '+' means concatenate.  

  _**WARNING! WARNING! WARNING!**_ You SHOULD re-calculate on your end the sha256 digest on each notification to check if it does match _sha256\_digest_. An attacker could POST or GET to your merchant server a false notification but he will never be able to compute a correct _sha256\_digest_. In _test_ mode _secret\_key_ is _134c8nfWK4XrrfkgN1UJ9ytqae76d7uDbs_.

* **attach**: free text field \(255 chars\) returned "as is" from step 1 for your own convenience.

Note: if you filled in the optional **merchant\_email** on step 1 you will receive the same information in an email plus possible alert email when - for any reason - if we were not able to notify you on **merchant\_return\_url** \(for example a server time out on your end\). We advise you to provide the **merchant\_email** in order to receive such alert emails. We strongly advise you to whitelist noreply@obyte-for-merchants.com

