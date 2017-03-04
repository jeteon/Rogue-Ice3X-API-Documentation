# Introduction

This is **not** the official documentation. Ice3X does not seem to care enough to maintain useful documentation of their API so this is unashamedly a hostile fork. Go [here](https://github.com/ICE3X/api-doc) if you want the official version. The alterations herein have been learned through the sweat and blood of many developers (most notably those in the credits) who reverse engineered the API. I pretty much went through the endpoints whilst writing a client of my own so most of the examples below should be representative.

Please feel free to add pull requests if you discover something I missed or something else undocumented about the API. 


# Overview

The [Ice3X trading platform](https://ice3x.com/) provides a simple API which allows for integration of business applications, payment systems, trading systems, mobile apps, etc. All requests use the `application/json` content type and must use HTTPS. Unless specified all API methods use HTTP POST.

If you have any feedback on the API please post a question in our support center.


## Getting started

I'm sure you've guessed by now that you'll be needing an API key. Log into your Ice3X account and then go to the [account page](https://ice3x.com/account). Just under the blue menu bar you'll see a link called "API Key". Click it. Then click the "Generate API key" button if details for one aren't already being shown to you. The API credentials will only be used for authenticated endpoints so feel free to follow along without taking the above step anyway if you're only interested in the public endpoints.

## Caveats

If this isn't your first rodeo, you'll know just how troublesome undocumented animals can be so keep the following in mind about the API:

- Many quantities are in [Satoshis](https://en.bitcoin.it/wiki/Satoshi_(unit)). Even fiat amounts may be in Satoshis so you'll have to multiply raw amounts by `1e8` and divide by that again unless you also internally work in Satoshis. Many...but not all. Sometimes they decide to use the raw value because its their API and they can if they want to.
- Requests and responses may look like normal JSON but for some reason the API cares about the order of keys so bear this in mind when developing your own code.
- Timestamps you come across are Unix timestamps that may or may not be in milliseconds. Also, where they appear, it is generally the string representation of the integer. Why not just milliseconds all the time? Because its more fun this way, silly.


## Things to bear in mind about this document

In all the sample outputs below, I've added whitespace to the responses to make them easier to read. Whitespace should generally not be added to request JSON strings, however, because it will affect value of the signature.


# Public API

The market data API is accessible through HTTPS and parameters are passed using `GET` requests. All responses are encoded in JSON. No authentication is required for calling market data API endpoints. Currently, the responses are cached for 30 seconds and all service calls are monitored for excessive load so please don’t query more often than once every minute.

## Tick

Basic summary data about a market.

Path: `/market/{instrument}/{currency}/tick`

Where `{instrument}` is one of "BTC" or "LTC". `{currency}` is generally "ZAR" but, suprisingly, can be "NGN", "USD" and "GBP" too.

Here's a sample response for [`/market/BTC/ZAR/tick`](https://api.ice3x.com/market/BTC/ZAR/tick):
```json
{
    "bestBid" :    17499.0, 
    "lastPrice" :  18100.0, 
    "instrument" : "BTC", 
    "timestamp" :  1488573548, 
    "currency" :   "ZAR", 
    "bestAsk" :    18099.0
}
```

Note: timestamp in seconds and prices are in the raw fiat amount.


## Orderbook

Get depth data about a market.

Path: `/market/{instrument}/{currency}/orderbook`

Here's a sample response for [`/market/BTC/ZAR/orderbook`](https://api.ice3x.com/market/BTC/ZAR/orderbook):
```json
{
    "currency" :   "ZAR",
    "instrument" : "BTC",
    "timestamp" :  1488573519,
    "asks" : [
        [18099.0, 0.015],
        [18100.0, 0.01049889],
        [18497.0, 0.10293],
        [18500.0, 0.1],
        [18864.0, 0.323206],
        [18875.0, 1.1715062],
        [20000.0, 0.06353688],
        [20850.0, 0.1],
        [21000.0, 0.0095]
    ],
    "bids" : [
        [17499.0, 0.00573272],
        [17499.0, 0.0057],
        [17499.0, 0.05654166],
        [17150.0, 0.028885],
        [17108.0, 0.25561],
        [17100.0, 0.00579005],
        [17000.0, 0.05101511],
        [16700.0, 0.10197369],
        [16300.0, 0.1509815],
        [15500.0, 0.02146527]
    ]
}
```

## Trades:

A list of the recent trades that have been executed on the market.

Path: `/market/{instrument}/{currency}/trades`.

Here's sample response from [`/market/BTC/ZAR/trades`](https://api.ice3x.com/market/BTC/ZAR/trades):
```json
[
  	{"tid" : 13659000, "amount" : 0.00258247, "price" : 18100.0, "date" : 1488569293},
  	{"tid" : 13658805, "amount" : 0.05211913, "price" : 18100.0, "date" : 1488566397},
  	{"tid" : 13657998, "amount" : 0.00275749, "price" : 18000.0, "date" : 1488550418},
  	{"tid" : 13657915, "amount" : 0.27227,    "price" : 18000.0, "date" : 1488549799},
  	{"tid" : 13657483, "amount" : 0.01477124, "price" : 18880.0, "date" : 1488542646},
  	{"tid" : 13657442, "amount" : 0.03131488, "price" : 18000.0, "date" : 1488541541},
  	{"tid" : 13657427, "amount" : 0.03,       "price" : 18000.0, "date" : 1488541492},
  	{"tid" : 13656900, "amount" : 0.0494036,  "price" : 17500.0, "date" : 1488529373},
  	{"tid" : 13656271, "amount" : 0.00579005, "price" : 17499.0, "date" : 1488524944},
  	{"tid" : 13655819, "amount" : 0.01090683, "price" : 17499.0, "date" : 1488519953}
]
```

You can also supply an optional parameter called `since` which will limit the returned response to trades before that ID. For instance:

````
 /market/BTC/ZAR/trades?since=59868345231
```

The observant among you may be wondering: "But master, how do I know whether a trade was a buy or sell?". Hehehe...I admire the spirit, grasshopper. You see, Ice3X provides no record of such things. All orders are equal before the eyes of the API.


# Private Endpoints

This section will describe the endpoints that are related to your account on Ice3X so they will all require authentication. The procedure will be described first and the endpoints themselves will follow thereafter.

## Authentication

To authenticate a request, you will need to build a string which includes several elements of the HTTP request. You will then use your API secret to calculate a [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) of that string. This will generate a signature string which needs to be added as a parameter of the request. The procedure will be described by walking through an example for the `/order/history` endpoint.

### Timestamp/nonce

A valid client timestamp must be used for authenticated requests. The timestamp included with an authenticated request must be within +/- 30 seconds of the server timestamp at the time when the request is received. Failing to submit a timestamp will result in authentication failure. The intention of these restrictions is to limit the possibility that intercepted requests could be replayed by an adversary.  

### Authentication example

Below is an example of creating signature for the `/order/history` API endpoint. In this example the parameters used to calculate the signature are:

 - API credentials:
    - API key:     `94f37c42-004d-11e7-82dc-3859f9c0fee0`
    - API secret:  `o5VbwsUU3b7GiJUuVGFVHLvnJ31bu/Bc2ndE6jdou539fQWmO87inWL1yb0gPVRHzQMrnxLFiFmTcAZj3ISuew==`
 
   These are not real so don't waste your time on any bright ideas.
 - URI: `/order/history`,
 - current timestamp in milliseconds: `1488579787907`,
 - request body: `{"currency":"ZAR", "instrument":"BTC", "limit":"10"}`

In general, the string to sign will be of the form:
```
{API endpoint}
{timestamp in milliseconds}
{request body}
```
This includes the newline characters (just `\n` not `\r\n`). For our example, the string that will be signed can be formed by running the following:

```
'/order/history' + '\n' + '1488579787907' + '\n' + '{"currency":"ZAR", "instrument":"BTC", "limit":"10"}'
```
 
When creating a signature for a `GET` request, the `POST` data will be empty and there is no need to add it to the string. The signature is found by calculating the [HMAC-SHA-512](https://tools.ietf.org/html/rfc4868) of the above string with the base64 decoded API secret as the key. The digest is then base64 encoded to get the signature. In pseudocode, it is the following series of steps:

```
key = base64_decode(API_SECRET)
str_to_sign = '/order/history' + '\n' + '1378818710123' + '\n' + '{"currency":"ZAR", "instrument":"BTC", "limit":"10"}'
raw_hmac = hmac_sha512(key, str_to_sign)
signature = base64_encode(raw_hmac)
```

If you've been following along diligently, the signature you should get is:
```
WtViNgP0eo8PoytY4d+wFu/nBXKvDw+fXIZdoC9GrNWsCQk5Om63iwszXHdx6srW8Cz3bwJaUwKHgBKLYdLzIQ==
```

Now we create a HTTP request with the following headers:

```http
"Accept":         "application/json"
"Accept-Charset": "UTF-8"
"Content-Type":   "application/json"
"apikey":         "94f37c42-004d-11e7-82dc-3859f9c0fee0"
"timestamp":      "1488579787907"
"signature":      "WtViNgP0eo8PoytY4d+wFu/nBXKvDw+fXIZdoC9GrNWsCQk5Om63iwszXHdx6srW8Cz3bwJaUwKHgBKLYdLzIQ=="
```

Don't forget to add the request body. Remember that the above is only an example. The `apikey`, `timestamp` and `signature` headers will need to be replaced by values that apply to your account and values that are calculated at the time when you make the request.


### Create an order

Path: `/order/create`.

This endpoint requires a `POST` request that has a JSON body with the following keys:
 - currency: generally "ZAR",
 - instrument: currently either "LTC" or "BTC",
 - price: the order price in Satoshis (an integer)
 - volume: order volume in Satoshis (an integer)
 - orderSide: either "Bid" or "Ask" depending on the order.
 - orderType: "Limit". I guess "Market" might work here too but I haven't tried it. I think if you need that functionality you can just submit a sell with a low price or a buy with a high one.
 - clientRequestId: No idea what this is. It might be a way to provide a custom label. The docs use "abc-cdf-1000". It gives whatever you put here back in the responses.

Here's a sample request body:
```json
{"currency":"ZAR","instrument":"BTC","price":13000000000,"volume":10000000,"orderSide":"Bid","ordertype":"Limit","clientRequestId":"abc-cdf-1000"}
```

Here's a sample success response:
```json
{
    "success" :         true,
    "errorCode" :       null,
    "errorMessage" :    null,
    "id" :              100,
    "clientRequestId" : "abc-cdf-1000"
}
```
That `id` is the order ID which might come in handy if you want to cancel it or something at a later stage.

And a sample error response:
```json
{
    "success" :        false, 
    "errorCode" :      3,
    "errorMessage" :   "Invalid argument.",
    "id" :             0,
    "clientRequestId": "abc-cdf-1000"
}
```

### Get Open Orders

Path: `/order/open`.

This endpoint requires a `POST` request that has a JSON body with the following keys:
 - currency: generally "ZAR",
 - instrument: currently either "LTC" or "BTC",
 - limit: the maximum number of orders to return
 - since: return only orders that are newer than the order with the given ID

You're probably tempted to think `limit` and `since` are optional parameters. Omit them at your peril for your offering to the API will not be deemed worthy. Use a `since` value of `1` if you prefer not to think about it.

A sample request body would be:
```json
{"currency":"ZAR","instrument":"BTC","limit":10,"since":99}
```

And a sample response:
```
{
    "errorCode": null, 
	"errorMessage": null, 
	"orders": [
		{
			"status":          "Placed",
			"clientRequestId": null, 
			"trades":          [], 
			"instrument":      "BTC", 
			"orderSide":       "Bid", 
			"ordertype":       "Limit", 
			"price":           13000000000, 
			"errorMessage":    null, 
			"creationTime":    1488582367361, 
			"volume":          1000000, 
			"currency":        "ZAR", 
			"openVolume":      1000000, 
			"id":              100
        }
	], 
	"success": true
}
```

It's good to see our order from before is in there, isn't it? Our `clientRequestId` from before seems to have fallen away though. Ah well.

Also, did you notice `creationTime` is in milliseconds? Just thought I'd point that out in case you were starting to think they always returned timestamps in seconds.


### Get Order History

Get a list of all your orders.

Path: `/order/history`.

This endpoint requires a `POST` request that has a JSON body with the following keys:
 - currency: generally "ZAR",
 - instrument: currently either "LTC" or "BTC",
 - limit: the maximum number of orders to return
 - since: return only orders that are newer than the order with the given ID

A sample request body would be:

```json
{"currency":"ZAR","instrument":"BTC","limit":10,"since":95}
```

sample response:

```json
{
	"errorCode": null, 
	"errorMessage": null, 
	"orders": [
		{
			"status":          "Placed", 
			"clientRequestId": null, 
			"ordertype":       "Limit", 
			"price":           13000000000, 
			"errorMessage":    null, 
			"volume":          1000000, 
			"currency":        "ZAR", 
			"openVolume":      1000000, 
			"id":              100, 
			"orderSide":       "Bid", 
			"trades":          [], 
			"creationTime":    1488582367361, 
			"instrument":      "BTC"
		},
		{
			"status":          "Fully Matched", 
			"clientRequestId": null, 
			"ordertype":       "Limit", 
			"price":           1750000000000, 
			"errorMessage":    null, 
			"volume":          4940360, 
			"currency":        "ZAR", 
			"openVolume":      0, 
			"id":              98, 
			"orderSide":       "Ask", 
			"trades": [
				{
					"fee":           29642, 
					"description":   null, 
					"price":         1750000000000, 
					"creationTime":  1488529372831, 
					"volume":        4940360, 
					"id":            124
				}
			], 
			"creationTime":    1488484667203, 
			"instrument":      "BTC"
		},
		{
			"status":          "Cancelled", 
			"clientRequestId": null, 
			"ordertype":       "Limit", 
			"price":           1655000000000, 
			"errorMessage":    null, 
			"volume":          13200967, 
			"currency":        "ZAR", 
			"openVolume":      13200967, 
			"id":              97, 
			"orderSide":       "Bid", 
			"trades":          [], 
			"creationTime":    1488460912354, 
			"instrument":      "BTC"
		}
	], 
	"success": true
}
```

I'm yet to catch a "Partially Matched" example "in the wild". If any of you do, please add it to the above response sample.


### Trade History

Path: `/order/trade/history`.

This endpoint requires a `POST` request that has a JSON body with the following keys:
 - currency: generally "ZAR",
 - instrument: currently either "LTC" or "BTC",
 - limit: the maximum number of trades to return
 - since: return only trades that are newer than the trade with the given ID

A sample request body would be:

```json
{"currency":"ZAR","instrument":"BTC","limit":10,"since":33434568724}
```

In return, the API might give you something like the following:
```
{
	"errorCode": null, 
	"errorMessage": null, 
	"success": true, 
	"trades": [
		{
			"fee": 			29642, 
			"description":	null, 
			"price": 		1750000000000, 
			"creationTime": 1488529372831, 
			"volume": 		4940360, 
			"id": 			124
		}
	]
}
```

You'll notice the trade is the same one included in the "Fully Matched" order from the `/order/history` call. That's pretty neat, IMHO. I have no idea what `description` could possibly include when it isn't `null` but my interest is definitely piqued.


### Order Detail

Path: `/order/detail`.

The `POST` body for this endpoint is a JSON body with a single key `orderIds` which is a list of order IDs to retrieve the details of. A sample request might look like the following:

```json
{"orderIds":[98,100]}
```

An in return the API would give:
```
{
	"success":true,
	"errorCode":null,
	"errorMessage":null,
	"orders":[
		{
			"id":			   100,
			"currency":		   "ZAR",
			"instrument":	   "BTC",
			"orderSide":	   "Bid",
			"ordertype":	   "Limit",
			"creationTime":	   1488582367361,
			"status":		   "Placed",
			"errorMessage":    null,
			"price":		   13000000000,
			"volume":	 	   1000000,
			"openVolume":	   1000000,
			"clientRequestId": null,
			"trades":		   []
		}
	]
}
```

Why did only one order come back even though we specified two order IDs and the reponse format suggests more than one order can come back? These are not things for us mere users to discern.


### Cancel an Order

Cancelling your order so soon? Very well then. Here's how.

Path: `/order/cancel`

The `POST` body for this request, has just one key `orderIds` which is a list of order IDs you would like to cancel. For instance, a sample request might look like the following:

```json
{"orderIds":[5,100]}
```

sample response:
```json
{
	"success" : true,
	"errorCode" : null,
	"errorMessage" : null,
	"responses" : [
		{"success" : false, "errorCode" : 3, "errorMessage" : "order does not exist.", "id" : 5},
		{"success" : true, "errorCode" : null,"errorMessage" : null, "id" : 100}
	]
}
```

I think its pretty cool they let you cancel a bunch all at once like that and give you feedback on each request individually.


### Fund Transfer API

 > [2015-06-20 01:59:05]: Coming soon

Ooooh... I can't wait!


# Troubleshooting

Unfortunately, the API seems to respond with `{"errorCode": 1, "errorMessage": "Authentication failed.", "success": false}` for a bunch of things that affect the authentication procedure so you have to try quite a few things to get to the bottom of it. There are a number of things I've come across so far that might lead to this:

 - your API credentials are incorrect ("Authentication failed" is actually accurate here). I'll say no more.
 - the API has a tolerance of only 30s either way for the timestamp value you use in authenticated requets. Check the API server's timestamp by looking at the `Date` header returned on the `/market/BTC/ZAR/tick` endpoint. This should be as close as possible to the value returned by running `date -u` on your system.
 - the keys in the JSON you're sending to the server are in the wrong order. I know it seems silly but for some reason the server seems to care. Check that each key in your request appears in the same order as above.
 - did you give a floating point value for a volume, price or timestamp?
 - when building the JSON bodies, did you provide values for **all** the keys? 
 
Please let me know if you come across any other things that helped you out in the past.


# Reference Clients

A number of reference clients were created by Ice3X. I can confirm the Go and C# clients work but your mileage may vary with the others:

- Java:   https://github.com/ICE3X/api-client-java
- Go:     https://github.com/ICE3X/api-client-go
- C#:     https://github.com/ICE3X/api-client-.net-C
- NodeJS: https://github.com/ICE3X/api-client-nodejs

Please let me know if you try any and they work for you or if you encounter others in the wild.

# Credits

A lot of what I have come to know about the API is due to the following people:
    - [https://github.com/martinbajalan](https://github.com/martinbajalan): He who started it [all](https://github.com/ICE3X/api-doc)...he who seems to have moved on.
    - [George Bence](https://bitcoinsouthafrica.slack.com/team/georgebence): First known human (at least to me) to have successfully used the API
    - [Zeptin](https://bitcoinsouthafrica.slack.com/team/zeptin): API whisperer and fellow OEM documentation decipherer
    - [Chris Naude](https://bitcoinsouthafrica.slack.com/team/bitflux): He who showed me the API was more than just a legend from the [ancient texts](https://ice3x.co.za/api/).

# Disclaimer

I'm in no way affiliated with Ice3X so I can't help you in any way with account queries, etc. I suggest you use the [support portal](http://support.ice3.com/support/home) or e-mail them on [support@ice3x.com](mailto:support@ice3x.com) for that. Since I don't work for them I also can't comment on the inner workings of the API beyond what is publicly observable. 
