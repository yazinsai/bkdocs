---
weight: 11
title: Real-time data
---

# Real-time data

Alpaca Data API v2 provides websocket streaming for trades, quotes and minute bars. This helps receive the most up to date market information that could help your trading strategy to act upon certain market movements.

Once a connection is established and you have successfully authenticated yourself you can subscribe to trades, quotes and minute bars for a particular symbol or multiple symbols.


## Subscription plans

The limitations listed below are default values but can be configured upon request.

**IEX**

* You can only connect to the IEX data source. One concurrent connection is allowed.
* Subscription is limited to 30 channels at a time for trades and quotes. 
* There is no limit for the number of channels with minute bars.
* Minute bars are based on the trades from IEX.


**SIP**

* There is no limit for the number of channels at a time for trades, quotes and minute bars.
* Trades, quotes and minute bars are direct feeds from the CTA (administered by NYSE) and UTP (administered by Nasdaq) SIPs.


## Common behavior

### URL
To access real-time data use the URL below, substituting `iex` or `sip` to `{source}` depending on your subscription.


`wss://stream.data.alpaca.markets/v2/{source}`


Attemption to access a data source not available for your subscription will result in an error during authentication.

### Message format
Every message you receive from the server will be in the format:


`[{"T": "{message_type}", {contents}},...]`


Control messages (i.e. where `"T"` is `error`, `success` or `subscription`) always arrive in arrays of size one to make their processing easier.

Data points however may arrive in arrays that have a length that is greater than one. This is to facilitate clients whose connection is not fast enough to handle data points sent one by one. Our server buffers the outgoing messages but slow clients may get disconnected if their buffer becomes full.


### Encoding and compression

Messages over the websocket are in encoded as clear text.

To reduce bandwidth requirements we have implemented compression as per [RFC-7692](https://tools.ietf.org/html/rfc7692). Our SDKs handle this for you so in most cases you won’t have to implement anything yourself.


## Communication flow

The communication can be thought of as two separate phases: **establishment** and **receiving data**.

### Establishment

To establish the connection **first you will need to connect** to our server using the URL above.

Upon successfully connecting, you will receive the welcome message:


`[{"T":"success","msg":"connected"}]`


Now you will need to **authenticate** yourself using your credentials by sending the following message:


`{"action": "auth", "key": "{KEY_ID}", "secret": "{SECRET}"}`


Please note that each account can have up to one concurrent websocket connection. Subsequent attempts to connect are rejected.

If you provided correct credentials you will receive another `success` message:


`[{"T":"success","msg":"authenticated"}]`


### Receiving data
Congratulations, you are ready to receive real-time market data!

You can send one or more subscription messages (described [below](https://alpaca.markets/docs/api-documentation/api-v2/market-data/alpaca-data-api-v2/real-time/#subscribe) and after confirmation you will receive the corresponding market data.

At any time you can subscribe to or unsubscribe from symbols. Please note that due to the internal buffering mentioned above for a short while you may receive data points for symbols you have recently unsubscribed from.

## Client to server

### Authentication
After connecting you will have to authenticate as described above.


`{"action":"auth","key":"CK******************","secret":"***********************************"}`


### Subscribe
You can subscribe to `trades`, `quotes` and `bars` of a particular symbol (or `*` for every symbol in the case of `bars`). A `subscribe` message should contain what subscription you want to add to your current subscriptions in your session so you don’t have to send what you’re already subscribed to.


`{"action":"subscribe","trades":["AAPL"],"quotes":["AMD","CLDR"],"bars":["AAPL","VOO"]}`


You can also omit either one of them (`trades`, `quotes` or `bars`) if you don’t want to subscribe to any symbols in that category but be sure to include at least one of the three.

Unsubscribe#
Much like `subscribe` you can also send an `unsubscribe` message that subtracts the list of subscriptions specified from your current set of subscriptions.


`{"action":"unsubscribe","trades":["VOO"],"quotes":["IBM"],"bars":[]}`


## Server to client

### Control messages
You may receive the following control messages during your session.


`[{"T":"success","msg":"connected"}]`


You have successfully connected to our server.


`[{"T":"success","msg":"authenticated"}]`


You have successfully authenticated.

### Errors
You may receive an error during your session. You can differentiate between them using the list below.


`[{"T":"error","code":400,"msg":"invalid syntax"}]`


The message you sent to the server did not follow the specification


`[{"T":"error","code":401,"msg":"not authenticated"}]`


You have attempted to subscribe or unsubscribe before authentication.


`[{"T":"error","code":402,"msg":"auth failed"}]`


You have provided invalid authentication credentials.


`[{"T":"error","code":403,"msg":"already authenticated"}]`


You have already successfully authenticated during your current session.


`[{"T":"error","code":404,"msg":"auth timeout"}]`


You failed to successfully authenticate after connecting. You have a few seconds to authenticate after connecting.


`[{"T":"error","code":405,"msg":"symbol limit exceeded"}]`


The symbol subscription request you sent would put you over the limit set by your subscription package. If this happens your symbol subscriptions are the same as they were before you sent the request that failed.


`[{"T":"error","code":406,"msg":"connection limit exceeded"}]`


You already have an ongoing authenticated session.


`[{"T":"error","code":407,"msg":"slow client"}]`


You may receive this if you are too slow to process the messages sent by the server. Please note that this is not guaranteed to arrive before you are disconnected to avoid keeping slow connections active forever.


`[{"T":"error","code":408,"msg":"v2 not enabled"}]`


The account does not have access to Data v2.


`[{"T":"error","code":409,"msg":"insufficient subscription"}]`


You have attempted to access a data source not available in your subscription package.


`[{"T":"error","code":500,"msg":"internal error"}]`


An unexpected error occurred on our end and we are investigating the issue.

**Subscription confirmation**

After subscribing or unsubscribing you will receive a message that describes your current list of subscriptions.


`[{"T":"subscription","trades":["AAPL"],"quotes":["AMD","CLDR"],"bars":["IBM","AAPL","VOO"]}]`


You will always receive your entire list of subscriptions, as illustrated by the sample communication excerpt below:

```json
> {"action": "subscribe", "trades": ["AAPL"], "quotes": ["AMD", "CLDR"], "bars": ["*"]}
< [{"T":"subscription","trades":["AAPL"],"quotes":["AMD","CLDR"],"bars":["*"]}]
> {"action": "unsubscribe", "bars": ["*"]}
> [{"T":"subscription","trades":["AAPL"],"quotes":["AMD","CLDR"],"bars":[]}]
```


## Data points

Multiple data points may arrive in each message received from the server. These data points have the following formats, depending on their type.

## Trade schema

| Attribute | Type       | Notes                  |
| --------- | ---------- | ---------------------- |
| `T`   | string | message type, always “t” |
| `S`     | string | symbol|
| `i`     | int | trade ID|
| `x`     | string    | exchange code where the trade occurred |
| `p`     | number | trade price|
| `s`     | int | trade size|
| `t`     | string | RFC-3339 formatted timestamp with nanosecond precision|
| `c`     | array<string> | trade condition|
| `z`     | string | tape |


### Example
```json
{
  "T": "t",
  "i": 96921,
  "S": "AAPL",
  "x": "D",
  "p": 126.55,
  "s": 1,
  "t": "2021-02-22T15:51:44.208Z",
  "c": [
    "@",
    "I"
  ],
  "z": "C"
}
```


## Quote schema

| Attribute | Type       | Notes                  |
| --------- | ---------- | ---------------------- |
| `T`   | string | message type, always “q” |
| `S`     | string | symbol|
| `ax`     | string | ask exchange code|
| `ap`     | number    | ask price |
| `as`     | int | ask size|
| `bx`     | string | bid exchange code|
| `bp`     | number    | bid price |
| `bs`     | int | bid size|
| `s`     | int | trade size|
| `t`     | string | RFC-3339 formatted timestamp with nanosecond precision|
| `c`     | array<string> | quote condition|
| `z`     | string | tape |


### Example
```json
{
  "T": "q",
  "S": "AMD",
  "bx": "U",
  "bp": 87.66,
  "bs": 1,
  "ax": "Q",
  "ap": 87.68,
  "as": 4,
  "t": "2021-02-22T15:51:45.335689322Z",
  "c": [
    "R"
  ],
  "z": "C"
}
```


## Bar schema

| Attribute | Type       | Notes                  |
| --------- | ---------- | ---------------------- |
| `T`   | string | message type, always “b” |
| `S`     | string | symbol|
| `o`     | number | open price|
| `h`     | number    | high price |
| `l`     | number | low price|
| `c`     | number | close price|
| `v`     | int    | volume |
| `t`     | string | RFC-3339 formatted timestamp|

### Example
```json
{
  "T": "b",
  "S": "SPY",
  "o": 388.985,
  "h": 389.13,
  "l": 388.975,
  "c": 389.12,
  "v": 49378,
  "t": "2021-02-22T19:15:00Z"
}
```


## Streaming example

```yaml
$ wscat -c wss://stream.data.alpaca.markets/v2/sip
connected (press CTRL+C to quit)
< [{"T":"success","msg":"connected"}]
> {"action": "auth", "key": "*****", "secret": "*****"}
< [{"T":"success","msg":"authenticated"}]
> {"action": "subscribe", "trades": ["AAPL"], "quotes": ["AMD", "CLDR"], "bars": ["*"]}
< [{"T":"t","i":96921,"S":"AAPL","x":"D","p":126.55,"s":1,"t":"2021-02-22T15:51:44.208Z","c":["@","I"],"z":"C"}]
< [{"T":"q","S":"AMD","bx":"U","bp":87.66,"bs":1,"ax":"X","ap":87.67,"as":1,"t":"2021-02-22T15:51:45.3355677Z","c":["R"],"z":"C"},{"T":"q","S":"AMD","bx":"U","bp":87.66,"bs":1,"ax":"Q","ap":87.68,"as":4,"t":"2021-02-22T15:51:45.335689322Z","c":["R"],"z":"C"},{"T":"q","S":"AMD","bx":"U","bp":87.66,"bs":1,"ax":"X","ap":87.67,"as":1,"t":"2021-02-22T15:51:45.335806018Z","c":["R"],"z":"C"}]
```

