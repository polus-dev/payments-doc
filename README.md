# payments-doc

This document searches for integration and interaction between merchants and [poluspay.com](https://poluspay.com) – explore our guide with examples and start accept crypto now! If you have any questions during the integration process, feel free to ask us for help [polus_alex](https://t.me/polus_alex)

## preparation

Create a new account at [app.poluspay.com](https://app.poluspay.com), then create a new merchant, specifying the name, domain name of your business website and other information. Next, you will need to confirm domain ownership by following the instructions and creating a TXT record for your domain.

After these steps set the URL for webhooks, you can view your `signing key` as well as generate an `api key`. Save both of keys.

> :warning: **Security**: We are not responsible if your keys are compromised!

## invoice creation

To invoice, send a HTTP POST request to `https://pay.polus.fi/api/v1/merchant/new-payment`  
and specify the body in JSON format with the following parameters:

| parameter     | type        | required  | description                                         |
| ------------- | ----------- | --------- | --------------------------------------------------- |
| `amount`      | string      | yes       | `Floating number limited to 4 decimal after point`  |
| `asset`       | string      | yes       | destination coin `usdt` \| `usdc` \| `weth` \|      |
| `description` | string      | yes       | invoice description from 0 to 256 symbols           |
| `address`     | string      | no        | replace the address from the merchant's settings    |
| `expiry_time` | integer     | no        | payment lifetime in seconds from 900 to 3600        |

The request must include the `X-Polus-Api-Token` header containing the `api key`.

**short example with curl**
```bash
curl https://pay.polus.fi/api/v1/merchant/new-payment \
        -X POST -H "X-Polus-Api-Token: {YOUR_API_KEY}" \
        -d '{"description": "Something cool stuff", "amount": "0.1", "asset": "usdc"}' | jq .
```

## webhooks receiving

In your personal account, you had the option to specify the path to receive server notifications(webhooks), which will be delivered to your server after each successful payment. You will receive HTTP POST requests with the following json body:

| index | parameter          | required  | description
| ----- | ------------------ | --------- | ----------
| 1.    | `payment_id`       | `yes`     | uuid of invoice
| 2.    | `created_at`       | `yes`     | unix timestamp
| 3.    | `merchant_address` | `yes`     | merchant evm address
| 4.    | `decimals_in`      | `yes`     | count of decimals after point for `token_in`
| 5.    | `decimals_out`     | `yes`     | count of decimals after point for `token_out`
| 6.    | `amount_in`        | `yes`     | amount paid by customer
| 7.    | `token_in`         | `yes`     | token evm address selected by customer
| 8.    | `amount_out`       | `yes`     | the amount the merchant received
| 9.    | `token_out`        | `yes`     | the token evm address of `amount_out`
| 10.   | `chain_id`         | `yes`     | chain id (1 - ethereum, 137 - polygon)
| -     | `signature`        | `yes`     | SHA512 HMAC signature encoded in hex

After receiving the webhook, you must to glue the received parameters by index from the table into a string with a `:` delimiter without `signature`: `"{payment_id}:{created_at}:{merchant_address}:{decimals_in}:{decimals_out}:{amount_in}:{token_in}:{amount_out}:{token_out}:{chain_id}"`. Next, use the `SHA512 HMAC` algorithm for this string along with the salt as your `signing key`. Next you **must**, compare the result of the hash function and the `signature` you got in the webhook.

> :warning: **Security**: If you don't verify the signature you run the risk of an attack on your server. Signature verification gives you reliable information about the authenticity of the request source!
