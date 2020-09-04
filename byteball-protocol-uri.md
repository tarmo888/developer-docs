---
description: >-
  This URI protocol enables developers to open the wallet app in desired screen
  with pre-filled inputs.
---

# URI protocol

If you want to open Obyte app for users then there is a `obyte:` protocol [URI ](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Examples)for that. If you are not aware what it is then it's like `mailto:` , `ftp:` , `tel:` or [intents on Android](https://developer.android.com/reference/android/content/Intent). When user visits a link with that protocol then it can open the wallet app in specific screen with pre-filled inputs, helping users to insert long wallet addresses or amounts without the need to copy-paste them.

## Opening Send screen

Following hyperlinks will open the wallet app on Send screen. These hyperlinks \(without the HTML\) also work when posted as chat messages.

### Requesting to send funds

We can [request payments with chat bot messages](payments/#requesting-payments) and we can use the same commands as links on the website. This is how it will look as a hyperlink:

```markup
<a href="obyte:WALLET_ADDRESS?amount=123000&amp;asset=base">
    open Send screen with bytes
</a>
```

`WALLET_ADDRESS` should be replaced with the wallet address where the funds should be sent, `amount` parameter should have `bigInt` type number in bytes \(not KByte, not MByte, not GByte\) and `asset` parameter is optional, by default it is `base` for bytes, but it can be issued [asset unit ID](issuing-assets-on-byteball.md#asset-id) too.

One website that heavily relies on this feature is [Currency converter for Obyte](https://tarmo888.github.io/bb-convert/) \([source code](https://github.com/tarmo888/bb-convert)\).

It is also possible to request that the payment to be done from specific address in user's wallet. This doesn't work as chat message, [request payment from single address instead](payments/#payments-from-single-address-wallet).

```markup
<a href="obyte:WALLET_ADDRESS?amount=123000&amp;from_address=USER_WALLET_ADDRESS">
    open Send screen with bytes
</a>
```

### Requesting to send data to AA

In order to open a Send screen with pre-filled payment and data, the data object needs to be converted into URI encoded Base64 \(using `JSON.stringify`, `btoa` and`encodeURIComponent`\) string. Autonomous agents can accept nested data, but wallet Send screen will only show the first level of the data.

```javascript
var base64 = global.btoa || require('btoa');
var data = {
    number: 1,
    foo: "bar",
    foobar: {
        foo: "bar",
    },
    "?<?>?": ["a",1,2,3],
};
var json_string = JSON.stringify(data);
var base64_string = base64(json_string);
var uri_encoded_base64data = encodeURIComponent(base64_string);
```

This will open Send screen, which will be pre-filled to post 10 000 bytes and data object.

```markup
<a href="obyte:AA_ADDRESS?amount=10000&amp;base64data=eyJudW1iZXIiOjEsImZvbyI6ImJhciIsImZvb2JhciI6eyJmb28iOiJiYXIifSwiPzw%2FPj8iOlsiYSIsMSwyLDNdfQ%3D%3D">
    open Send screen with bytes and data
</a>
```

### Requesting to post data

This will open Send screen, which will be pre-filled to post unstructured key/value pairs as data \(single-address wallet required\). Keys need to be unique.

```markup
<a href="obyte:data?app=data&amp;anykey1=anyvalue&amp;anykey2=anyvalue">
    open Send screen with data
</a>
```

### Requesting to post data feed

This will open Send screen, which will be pre-filled to post indexable key/value pairs as data feed \(single-address wallet required\). Keys need to be unique.

```markup
<a href="obyte:data?app=data_feed&amp;anykey1=anyvalue&amp;anykey2=anyvalue">
    open Send screen with data feed
</a>
```

### Requesting to post a profile

This will open Send screen, which will be pre-filled to post key/value pairs as profile info \(single-address wallet required\). Keys need to be unique.

```markup
<a href="obyte:data?app=profile&amp;anykey1=anyvalue&amp;anykey2=anyvalue">
    open Send screen with profile
</a>
```

### Requesting to post attestation

This will open Send screen, which will be pre-filled to post key/value pairs as someone else attestation profile \(single-address wallet required\). Keys need to be unique.

```markup
<a href="obyte:data?app=attestation&amp;address=WALLET_ADDRESS&amp;anykey1=anyvalue&amp;anykey2=anyvalue">
    open Send screen with attestation
</a>
```

### Requesting to post a poll

This will open Send screen, which will be pre-filled to post poll \(single-address wallet required\). Option keys can be named anything, but values need to be unique. Question parameter is also required.

```markup
<a href="obyte:data?app=poll&amp;question=How%20are%20you&amp;option1=fine&amp;option2=tres%20bien">
    open Send screen with poll
</a>
```

### Requesting to post a AA definition

This will open Send screen, which will be pre-filled to post Autonomous Agent definition \(single-address wallet required\). The `definition` parameter needs to be URL encoded.

```markup
<a href="obyte:data?app=definition&amp;definition=%7B%0A%09messages%3A%20%5B%0A%09%09%7B%0A%09%09%09app%3A%20'payment'%2C%0A%09%09%09payload%3A%20%7B%0A%09%09%09%09asset%3A%20'base'%2C%0A%09%09%09%09outputs%3A%20%5B%0A%09%09%09%09%09%7B%0A%09%09%09%09%09%09address%3A%20%22%7Btrigger.address%7D%22%0A%09%09%09%09%09%7D%0A%09%09%09%09%5D%0A%09%09%09%7D%0A%09%09%7D%0A%09%5D%0A%7D">
    open Send screen with AA definition
</a>
```

Because most browsers support only URLs up to 2048 characters length, it is also possible to post a AA definition by making it fetch the definition from some other other address like this:

```markup
<a href="obyte:data?app=definition&amp;definition=https%3A%2F%2Fexample.com%2Fdefinition.json">
    open Send screen with AA definition
</a>
```

### Requesting to post short text content

This will open Send screen, which will be pre-filled to post short text content \(single-address wallet required\). The `content` parameter needs to be URL encoded and can be maximum of 140 chars before encoding.

```markup
<a href="obyte:data?app=text&amp;content=Lorem%20ipsum%20dolor%20sit%20amet%2C%20consectetur%20adipiscing%20elit.%0AQuisque%20venenatis%20elementum%20dolor%20in%20malesuada.%20Cras%20ultrices%20ultrices%20fermentum.%20Nullam%20ac%20commodo%20ante.%20Pellentesque%20quis%20nibh%20laoreet%2C%20sagittis%20augue%20in%2C%20dapibus%20nunc.%20Etiam%20vel%20eleifend%20urna%2C%20in%20tincidunt%20turpis%20nullam.">
    open Send screen with text content
</a>
```

## Receiving textcoins via link

Textcoins are funds that can be sent via text messages. These text messages contain 12 randomly picked words and by entering them into the wallet app in exact same order, user will get the funds that were added to them. If you have ever received any textcoins then you also know that you don't actually have to re-type or copy-paste those words, textcoin receivers are usually directed to [Obyte textcoin claiming page](https://byteball.org/#textcoin?test-test-test-test-test-test-test-test-test-test-test-test), which has green "Recieve funds" button, which will launch the wallet app with exactly those textcoin words. Obyte textcoin claiming page tries to check first if user has wallet installed, but basically, underneath that button is a hyperlink similar to this:

```markup
<a href="obyte:textcoin?RANDOMLY-GENERATED-TWELVE-WORDS">
    Recieve funds
</a>
```

So, additionally to [sending textcoins with a bot](payments/textcoins.md), you can also make clickable textcoins on your website \(link either to textcoin claiming page or directly to `obyte:` protocol\). Just make sure each user on your webiste has access to textcoins specifically generated for them. Otherwise, if all users see the same textcoins then it can become a rush to who claims the textcoins first.

## Interacting with chat bot

Bot Store on Obyte is like Apple App Store or Google Play Store, but instead of apps, Bot Store has chat bots that are written for Obyte platform. Each Obyte Hub can have each own bots that are shown for users of that app. Currently, most users are using the official wallet app and official Hub, but it makes sense that if somebody forks the wallet, they would use their own Hub for it too. This means that in the future, in order to get your chat bot exposed to maximum amount of users, it needs to get listed on all Hubs.

### Pairing with another device

Luckily, adding chat bots to your wallet app is much easier on Obyte than it is to side-load apps on iOS or Android. All you need to do is add the pairing code that bot outputs to your website - everyone who has Byteball app already installed and clicks on it, will get automatically paired with your chat bot. Hyperlink to pairing code would be something like this:

```markup
<a href="obyte:AnpzF9nVTV5JZXzlG2fSnA+8UmjFuBdbqU+rJchz3qcN@byteball.org/bb#0000">
    Add Sports Betting bot
</a>
```

One example, where all the official Hub bots are displayed on a website can be seen on [Bots section obyte.io](https://obyte.io/bots).

Pairing codes in hyperlinks are not limited to only chat bots, you could create a pairing link to any device, all you need to do is find you pairing invitation code \(Chat &gt; Add new device &gt; Invite other device\) and add the code after `obyte:` protocol inside the hyperlink.

### Sending commands to chat bots

Once you have paired with a chat bot already, you might wonder whether it is possible to get users from your website to some specific state in chat bot \(for example, by [sending a predefined command](./#predefined-chat-commands)\) with a click on any hyperlink. There are 2 options how to do that, first option would be to fill the `pairing_secrets` database table with all the possible commands, but if there could be indefinite number of valid commands then easiest would be to accept any pairing secret and use them as custom commands.

{% code title="conf.js" %}
```javascript
exports.permanent_pairing_secret = '*';
```
{% endcode %}

Websites that use this feature are [BB Odds](https://bb-odds.herokuapp.com/) and [Polls section on obyte.io](https://obyte.io/polls). How it can be done can be seen from Poll bot \([source code](https://github.com/byteball/poll-bot/blob/master/poll-bot.js)\). For example, opening a poll app with results of specific poll can be done with hyperlink like this:

```markup
<a href="obyte:AhMVGrYMCoeOHUaR9v/CZzTC34kScUeA4OBkRCxnWQM+@byteball.org/bb#stats-HAXKXC1EBn8GtiAiW0xtYLAJiyV4V1jTt1rc2TDJ7p4=">
    see stats
</a>
```

Steem Attestation bot uses this method also as an alternative way how to refer other people, source code for that is more advanced that the poll bot code, but [it can be found on Github](https://github.com/byteball/steem-attestation/blob/66e72d4062f7c1f5a8a057366023c5eaa6863bf4/attestation.js#L90).

## Using protocol URI in QR code

`obyte:` protocol is not only for websites, it could be used in physical world too with QR code. This means you if users have Obyte app installed and they scan any QR code that contains any of the above codes \(just value of the href, without quotes and without the HTML around them\) then installed Obyte app will open the same way. QR code is not just meant to be printed on paper, it can work on websites too, giving the users the ability to use your services cross-device \(browse the website on laptop, scan QR code with phone and complete payment on phone\).

There are many online QR code generators to create those QR codes manually, but QR codes can be created in real-time too. Following is the example how to create a QR code with `obyte:` protocol link using jQuery.

```markup
<div class="container">
	<div id="qr_code"></div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.3.1/jquery.min.js"
	integrity="sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8="
	crossorigin="anonymous"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery.qrcode/1.0/jquery.qrcode.min.js"
	integrity="sha256-9MzwK2kJKBmsJFdccXoIDDtsbWFh8bjYK/C7UjB1Ay0="
	crossorigin="anonymous"></script>
<script type="text/javascript">
$(document).ready(function() {
	$('#qr_code').html('').qrcode({
		width: 512,
		height: 512,
		text: 'obyte:WALLET_ADDRESS?amount=123000&asset=base'
	});
});
</script>
```

A website that creates QR codes this way is [Currency converter for Obyte](https://tarmo888.github.io/bb-convert/) \([source code](https://github.com/tarmo888/bb-convert)\) and Steem Attestation bot \([source code](https://github.com/byteball/steem-attestation/blob/master/public/qr/index.html)\) generates referral links this way. Alternatively, there is also code example [how to generate QR code like that on server side](tutorials-for-newcomers/log-in-on-website-with-byteball.md).

## Using these with testnet wallets

If you are testing these functions with [TESTNET headless wallet](https://github.com/byteball/headless-obyte#testnet) and [TESTNET GUI wallet](https://obyte.org/testnet.html) then instead of `obyte:` protocol, use `obyte-tn:`. This can be easily controlled in your bot's code like this:

```javascript
const pairingProtocol = process.env.testnet ? 'obyte-tn:' : 'obyte:';
```

