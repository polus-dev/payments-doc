# DEPRECATED 
# Actual version https://docs.poluspay.com/

This document searches for integration and interaction between merchants and [poluspay.com](https://poluspay.com) – explore our guide with examples and start accept crypto now! If you have any questions, feel free to ask us for help [polus_sergey](https://t.me/polus_sergey)

## preparation

Create a new account at [app.poluspay.com](https://app.poluspay.com), then create a new merchant, specifying the name, domain name of your business website and other information. Next, you will need to confirm domain ownership by following the instructions and creating a TXT record for your domain.

After these steps set the URL for webhooks, you can view your `signing key` as well as generate an `api key`. Save both of keys.

> :warning: **Security**: We are not responsible if your keys are compromised!

## invoice creation

To invoice, send a HTTP POST request to `https://pay.polus.fi/api/v1/payment/create`  
and specify the body in JSON format with the following parameters:

| parameter     | type        | required  | description                                         |
| ------------- | ----------- | --------- | --------------------------------------------------- |
| `amount`      | string      | `yes`     | Floating number limited to 4 decimal after point    |
| `asset`       | string      | `yes`     | destination coin `usdt` \| `usdc` \| `weth` \|      |
| `description` | string      | `yes`     | invoice description from 0 to 256 symbols           |
| `address`     | string      | `no`      | replace the address from the merchant's settings    |
| `expiry_time` | integer     | `no`      | payment lifetime in seconds from 900 to 3600        |

The request must include the `X-Polus-Api-Token` header containing the `api key`.

**short example with curl**
```bash
curl https://pay.polus.fi/api/v1/payment/create \
        -X POST -H "X-Polus-Api-Token: {YOUR_API_KEY}" \
        -d '{"description": "Something cool stuff", "amount": "0.1", "asset": "usdc"}' | jq .
```
```json
{
  "status": "ok",
  "result": {
    "payment_link": "https://pay.polus.fi/pay?id=9749a6a9-94b4-4fd0-ad5c-1701f7ca910b",
    "payment_id": "9749a6a9-94b4-4fd0-ad5c-1701f7ca910b"
  }
}
```

## webhooks receiving

In your personal account, you had the option to specify the path to receive server notifications(webhooks), which will be delivered to your server after each successful payment. You will receive HTTP POST requests with the following json body:

| index | parameter          | optional | description
| ----- | ------------------ | -------- | ----------
| 1.    | `payment_id`       | `no`     | uuid of invoice
| 2.    | `created_at`       | `no`     | unix timestamp
| 3.    | `merchant_address` | `no`     | merchant evm address
| 4.    | `decimals_in`      | `no`     | count of decimals after point for `token_in`
| 5.    | `decimals_out`     | `no`     | count of decimals after point for `token_out`
| 6.    | `amount_in`        | `no`     | amount paid by customer
| 7.    | `token_in`         | `no`     | token evm address selected by customer
| 8.    | `amount_out`       | `no`     | the amount the merchant received
| 9.    | `token_out`        | `no`     | the token evm address of `amount_out`
| 10.   | `chain_id`         | `no`     | chain id (1 - ethereum, 137 - polygon)
| -     | `signature`        | `no`     | SHA512 HMAC signature encoded in hex

After receiving the webhook, you must to glue the received parameters by index from the table into a string with a `:` delimiter without `signature`: `"{payment_id}:{created_at}:{merchant_address}:{decimals_in}:{decimals_out}:{amount_in}:{token_in}:{amount_out}:{token_out}:{chain_id}"`. Next, use the `SHA512 HMAC` algorithm for this string along with the salt as your `signing key`. Next you **must**, compare the result of the hash function and the `signature` you got in the webhook.

> :warning: **Security**: If you don't verify the signature you run the risk of an attack on your server. Signature verification gives you reliable information about the authenticity of the request source!

**example of webhook json body**
```json
{
  "payment_id": "db6be43e-7e18-4328-9476-c65becc48750",
  "created_at": 1672770679,
  "merchant_address": "0x1E039FafDE1A7Be16adF4612F7C52aa208702c57",
  "decimals_in": 6,
  "amount_in": "10000",
  "token_in": "USDC",
  "decimals_out": 6,
  "amount_out": "9700",
  "token_out": "USDC",
  "chain_id": 137,
  "signature": "38b2d54529f26545529338856c175a938c41e6881e79769d71e8abb72ea657b37fe5ac7dbba11e9ea4da096070bfd0c2e5ddb3fc587a330973b43a9658ff7630"
}
```
