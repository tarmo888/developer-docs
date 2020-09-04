---
description: >-
  Textcoins are a handy way of transfering money and asset to other people, in a
  text form.
---

# Textcoins

Textcoin is a payment sent though email or other text based media, such as chat apps. Textcoins can also be printed on paper, scratch cards, or PIN envelopes and passed on like paper wallets.

### Sending textcoins with bot

Sending textcoins from server-side \(headless\) wallets is easy: you use the same functions you normally use to send to Obyte addresses but instead of the recipient's Obyte address you write:

* `textcoin:userEmailAddress` for sending to email, or
* `textcoin:someUniqueIdentifierOtherThanEmail` for sending via any other text based media

Examples: `textcoin:pandanation@wwfus.org` or `textcoin:ASDFGHJKL`.

Sample code:

```javascript
var headlessWallet = require('headless-obyte');

let opts = {
    asset: null, 
    amount: 1000, // bytes
    to_address: "textcoin:pandanation@wwfus.org",
    email_subject: "Payment in textcoin"
};

headlessWallet.issueChangeAddressAndSendMultiPayment(opts, (err, unit, assocMnemonics) => {
    ....
});
```

### Sending textcoin emails

The `opts` object has an optional field `email_subject` which is the subject of the email sent to the user. To send emails, you need to set the following fields in your [configuration files](https://github.com/byteball/byteballcore#configuring):

* `from_email`: the address your emails are sent from. Must be on your domain, otherwise the emails are likely to be non-delivered or labeled as spam
* `smtpTransport`: type of SMTP transport used to deliver emails:
  * `local`: \(this is the default\) use `/usr/sbin/sendmail`
  * `direct`: connect directly to the recipient's MX server, this minimizes the number of intermediaries who are able to see and potentially steal the textcoin, but this method is not reliable as there are no retries
  * `relay`: send through a relay server. In this case, you need to also set its host name in `smtpRelay`. If it requires authentication, you need to also set `smtpUser` and `smtpPassword`.

### Callback response

The callback of `issueChangeAddressAndSendMultiPayment` and all similar functions has an optional 3rd parameter `assocMnemonics` which is an associative array of all generated textcoins \(BIP39 mnemonics\) indexed by the `address` from the input parameters.

```javascript
{
    "textcoin:pandanation@wwfus.org": "barely-album-remove-version-endorse-vocal-weasel-kitten-when-fit-elbow-crop"
}
```

All the keys of the array have a `textcoin:` prefix, like in input parameters.

If you are sending though any media other than email, you use `assocMnemonics` to find mnemonics generated for each output address \(in the example above, you would find it by the key `textcoin:ASDFGHJKL`\) and then deliver them yourself.

For example, if you are writing a Telegram bot that rewards users for certain actions, you create a clickable link by prefixing the mnemonic with `https://obyte.org/#textcoin?`:

```text
https://obyte.org/#textcoin?barely-album-remove-version-endorse-vocal-weasel-kitten-when-fit-elbow-crop
```

and send it in chat.

### More resources

If you are going to print a paper wallet from an ATM, you just print the mnemonic and optionally [encode textcoin words](../byteball-protocol-uri.md#receiving-textcoins-via-link) in the QR code. You could create your own [QR code generator on website](../byteball-protocol-uri.md#using-protocol-uri-in-qr-code) too with jQuery.

More about textcoins in Obyte blog  
[https://medium.com/obyte/sending-cryptocurrency-to-email-5c9bce22b8a9](https://medium.com/obyte/sending-cryptocurrency-to-email-5c9bce22b8a9)

Example code in a Telegram bot that sends users textcoins for passing a quiz: [https://github.com/byteball/telegram-quiz/blob/master/src/wallet.js](https://github.com/byteball/telegram-quiz/blob/master/src/wallet.js)

Example code that creates a list of textcoins for subsequent mass sending \(e.g. [via MailChimp](https://medium.com/byteball-help/using-mailchimp-to-mass-send-payments-as-textcoins-5c1db06342e3)\): [https://github.com/byteball/headless-obyte/blob/master/tools/create\_textcoins\_list.js](https://github.com/byteball/headless-obyte/blob/master/tools/create_textcoins_list.js)

