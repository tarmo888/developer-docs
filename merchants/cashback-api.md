---
description: >-
  This API is for merchants and payment processors who participate in the Obyte
  cashback program.
---

# Cashback API

This is API documentation to integrate cashback to your point of sale device, but you could also [build your own cashback app](https://github.com/myactivities/byteball-cashback-app), where this API has already been implemented. The first thing you need [sign up to Cashback Program](https://medium.com/obyte/byteball-cashback-program-9c717b8d3173) in order to get partner name and partner key.

After you accepted payment from the customer, you use this API to request cashback for your customer. You send a POST request to:

`https://byte.money/new_purchase`

\(for testnet use `https://byte.money/test/new_purchase`\)

and pass the following parameters:

* `partner`: \(string\) your partner name you were assigned when you signed up for the program.
* `partner_key`: \(string\) partner key used to authenticate your requests.
* `customer`: \(string\) any id of the customer who made a purchase such as customer name or id in your database. This field must be the same when the same customer makes another purchase.
* `order_id`: \(string\) unique id of the order associated with the purchase. You should never request cashback for the same order id again.
* `description`: \(string\) description of the purchase.
* `merchant`: \(string\) this field is used by payment processors only to identify the individual merchant. When another purchase is made from the same merchant, this fiels should be the same.
* `address`: \(string\) Wallet address or email address of the customer. The cashback will be sent to this address. If it is email address, a [textcoin](https://medium.com/byteball/sending-cryptocurrency-to-email-5c9bce22b8a9) will be sent. If the textcoin is not claimed within 1 week, it will be taken back.
* `currency`: \(string\) currency of the purchase. Supported values: USD, EUR, RUR, GBYTE, BTC. If the purchase was paid in any other currency, you should convert its amount to any of the supported currencies except GBYTE.
* `currency_amount`: \(number\) amount of the purchase in `currency`.
* `partner_cashback_percentage`: \(number\) the percentage of the amount you want to pay to the customer out of your own funds in addition to the regular cashback. Obyte will add the same percentage out of the distribution fund \(merchant match\). Default it 0. You have to deposit the funds in advance in order to fund this option.
* `purchase_unit`: \(string\) unit \(transaction id\) of the customer's purchase if it was paid in GBYTE.

### Response format

The response is in JSON format. It always contains a `result` field, which is either 'ok' or 'error'.

If the cashback was successfully sent to the customer, the response has `"result": "ok"` and also indicates the cashback amount and the unit it was sent in:

```javascript
{
    "result": "ok",
    "cashback_amount": 101318,
    "unit": "kYbc5KGKB/KnjynYePEF4cpumKmOz22xVKSVWbmtZ0M="
}
```

In case of an error, the response has `"result": "error"` and error description in `error` field:

```javascript
{
    "result": "error",
    "error": "authentication failed"
}
```

