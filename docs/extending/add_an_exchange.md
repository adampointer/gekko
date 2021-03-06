# Exchanges

*This is a technical document about adding a new exchange to Gekko.*

Gekko arranges all communication about when assets need to be bought or sold between the *strategy* and *Gekko Broker*. All differences between the different API's are abstracted away just below Gekko Broker inside an "exchange wrapper". This document describes all requirements for adding a new exchange wrapper (adding exchange support to Gekko).

## Gekko's expectations

When you add a new exchange to Gekko you need to expose an object that has methods to query the exchange. This exchange file needs to reside in `gekko/exchange/wrappers` and the filename is the slug of the exchange name + `.js`. So for example the exchange for Bitstamp is explained in `gekko/exchange/wrappers/poloniex.js`.

It is advised to use a npm module to query an exchange. This will seperate the abstract API calls from the Gekko specific stuff (In the case of Bitstamp there was no module yet, so I [created one](https://www.npmjs.com/package/bitstamp)).

Finally Gekko needs to know how it can interact with the exchange, so add a static method `getCapabilities()` that returns it's properties. The meaning of the properties are described [here](#capabilities).

## Gekko Broker's expectations

*If this documentation is not clear it please look at the examples in `gekko/exchange/wrappers/`.*

Gekko Manager implements an exchange like so:

    var Exchange = require('./wrapper' + [exchange slug]);
    this.exchange = new Exchange({key: '', secret: '', username: '', currency: 'USD', asset: 'BTC'});

It will run the following methods on the exchange object:

### getTicker

    this.exchange.getTicker(callback)

The callback needs to have the parameters of `err` and `ticker`. Ticker needs to be an object with atleast the `bid` and `ask` property in float.

### getFee

    this.exchange.getFee(callback)

The callback needs to have the parameters of `err` and `fee`. Fee is a float that represents the amount the exchange takes out of the orders Gekko places. If an exchange has a fee of 0.2% this should be `0.0002`.

### getPortfolio

    this.exchange.getPortfolio(callback)

The callback needs to have the parameters of `err` and `portfolio`. Portfolio needs to be an array of all currencies and assets combined in the form of objects, an example object looks like `{name: 'BTC', amount: 1.42131}` (name needs to be an uppercase string, amount needs to be a float).

### getLotSize

    this.exchange.getLotSize(tradeType, amount, size, callback)

The callback needs to have the parameters of `err` and `lot`. Lot needs to be an object with `amount` and `purchase` size appropriately for the exchange. In the event that the lot is too small, return 0 to both fields and this will generate a lot size warning in the portfolioManager.

Note: This function is currently optional. If not implemented `portfolioManager` will fallback to basic lot sizing mechanism it uses internally. However exchanges are not all the same in how rounding and lot sizing work, it is recommend to implement this function.

### buy

    this.exchange.buy(amount, price, callback);

*Note that this function is a critical function, retry handlers should abort quickly if attemps to dispatch this to the exchange API fail so we don't post out of date orders to the books.*

### sell

    this.exchange.sell(amount, price, callback);

This should create a buy / sell order at the exchange for [amount] of [asset] at [price] per 1 asset. If you have set `direct` to `true` the price will be `false`. The callback needs to have the parameters `err` and `order`. The order needs to be something that can be fed back to the exchange to see wether the order has been filled or not.

*Note that this function is a critical function, retry handlers should abort quickly if attemps to dispatch this to the exchange API fail so we don't post out of date orders to the books.*

### getOrder

    this.exchange.getOrder(order, callback);

Will only be called on orders that have been completed. The order will be something that the manager previously received via the `sell` or `buy` methods. The callback should have the parameters `err` and `order`. Order is an object with properties `price`, `amount` and `date`. Price is the (volume weighted) average price of all trades necesarry to execute the order. Amount is the amount of currency traded and Date is a moment object of the last trade part of this order.

### checkOrder

    this.exchange.checkOrder(order, callback);

The order will be something that the manager previously received via the `sell` or `buy` methods. The callback should have the parameters `err` and a `result` object. The result object will have two or three properties:

- `open`: whether this order is currently in the orderbook.
- `completed`: whether this order has executed (filled completely).
- `filledAmount`: the amount of the order that has been filled. This property is only needed when both `open` is true and `completed` is false.

### cancelOrder

    this.exchange.cancelOrder(order, callback);

The order will be something that the manager previously received via the `sell` or `buy` methods. The callback should have the parameterers `err` and `filled`, `filled` last one should be true if the order was filled before it could be cancelled.

## Trading method's expectations

The trading method analyzes exchange data to determine what to do. The trading method will also implement an exchange and run one method to fetch data:

### getTrades

    this.watcher.getTrades(since, callback, descending);

If since is truthy, Gekko requests as much trades as the exchange can give (up to ~10,000 trades, if the exchange supports more you can [create an importer](../features/importing.md)).

The callback expects an error and a `trades` object. Trades is an array of trade objects in chronological order (0 is older trade, 1 is newer trade). Each trade object needs to have:

- a `date` property (unix timestamp in either string or int)
- a `price` property (float) which represents the price in [currency] per 1 [asset].
- an `amount` proprty (float) which represent the amount of [asset].
- a `tid` property (float) which represents the tradeID.

## Error handling

It is the responsibility of the wrapper to handle errors and retry the call in case of a temporary error. Gekko exposes a retry helper you can use to implement an exponential backoff retry strategy. Your wrapper does need pass a proper error object explaining whether the call can be retried or not. If the error is fatal (for example private API calls with invalid API keys) the wrapper is expected to upstream this error. If the error is retryable (exchange is overloaded or a network error) an error should be passed with the property `notFatal` set to true. If the exchange replied with another error that might be temporary (for example an `Insufficiant funds` erorr right after Gekko canceled an order, which might be caused by the exchange not having updated the balance yet) the error object can be extended with an `retry` property indicating that the call can be retried for n times but after that the error should be considered fatal.

For implementation refer to the bitfinex implementation, in a gist this is what a common flow looks like:

- (async call) `exchange.buy`, then
- handle the response and normalize the error so the retry helper understands it, then
- the retry helper will determine whether the call needs to be retried, then
    - based on the error it will retry (nonFatal or retry)
    - if no error it will pass it to your handle function that normalizes the output

## Capabilities

Each exchange *must* provide a `getCapabilities()` static method that returns an object with these parameters:

- `name`: Proper name of the exchange
- `slug`: slug name of the exchange (needs to match filename in `gekko/exchanges/`)
- `currencies`: all the currencies supported by the exchange implementation in gekko.
- `assets`: all the assets supported by the exchange implementation in gekko.
- `pairs`: all allowed currency / asset combinations that form a market
- `maxHistoryFetch`: the parameter fed to the getTrades call to get the max history.
- `providesHistory`: If the getTrades can be fed a since parameter that Gekko can use to get historical data, set this to:
    - `date`: When Gekko can pass in a starting point in time to start returning data from.
    - `tid`: When Gekko needs to pass in a trade id to act as a starting point in time.
    - `false`: When the exchange does not support to give back historical data at all.
- `fetchTimespan`: if the timespan between first and last trade per fetch is fixed, set it here in minutes.
- `tradable`: if gekko supports automatic trading on this exchange.
- `requires`: if gekko supports automatic trading, this is an array of required api credentials gekko needs to pass into the constructor.
- `forceReorderDelay`: if after canceling an order a new one can't be created straight away since the balance is not updated fast enough, set this to true (only required for exchanges where Gekko can trade).
- `gekkoBroker`: set this to 0.6 for now, it indicates the version of Gekko Broker this wrapper is compatible with.

Below is a real-case example how `bistamp` exchange provides its `getCapabilities()` method:

```
Trader.getCapabilities = function () {
  return {
    name: 'Bitstamp',
    slug: 'bitstamp',
    currencies: ['USD', 'EUR'],
    assets: ['BTC', 'EUR'],
    maxTradesAge: 60,
    maxHistoryFetch: null,
    markets: [
      { pair: ['USD', 'BTC'], minimalOrder: { amount: 1, unit: 'currency' } },
      { pair: ['EUR', 'BTC'], minimalOrder: { amount: 1, unit: 'currency' } },
      { pair: ['USD', 'EUR'], minimalOrder: { amount: 1, unit: 'currency' } }
    ],
    requires: ['key', 'secret', 'username'],
    fetchTimespan: 60,
    tid: 'tid'
  };
}
```
