# Coinbase

An easy way to use the [Coinbase API](https://coinbase.com/docs/api/overview) and integrate bitcoin payments into your application.

This gem uses the [api key authentication method](https://coinbase.com/docs/api/overview).  If you would like to do an OAuth2 integration instead, you may want to start with the [OAuth2 Ruby Gem](https://github.com/intridea/oauth2) and use this gem as a reference.

## Installation

Add this line to your application's Gemfile:

    gem 'coinbase'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install coinbase

## Usage

Start by [enabling an API Key on your account](https://coinbase.com/account/integrations).

Next you can create an instance of the client and pass it your API Key as the first (and only) parameter.

```ruby
coinbase = Coinbase::Client.new ENV['COINBASE_API_KEY']
```

Notice here that we did not hard code the API key into our codebase, but set it in an environment variable instead.  This is just one example, but keeping your credentials separate from your code base is a good [security practice](https://coinbase.com/docs/api/overview#security).

Now you can call methods on `coinbase` similar to the ones described in the [api reference](https://coinbase.com/api/doc).  For example:

```ruby
coinbase.balance
=> #<Money fractional:20035300000 currency:BTC>
coinbase.balance.format
=> "200.35300000 BTC"
coinbase.balance.to_f
=> 200.353
```

Money amounts (USD, EUR, BTC, etc) are returned as [ruby money](https://github.com/RubyMoney/money) objects (integer amounts paired with a currency ISO code).  You can call `to_f`, `format`, or perform math operations on these objects.

You can see a list of [supported currencies here](https://github.com/coinbase/coinbase-ruby/blob/master/supported_currencies.json).

## Examples

### Check your balance

```ruby
coinbase.balance.to_f
=> 200.353 # BTC amount
```

### Send bitcoin

```ruby
r = coinbase.send_money 'user@example.com', 1.23
r.success?
=> true
r.transaction.status
=> 'pending' # this will change to 'complete' in a few seconds if you are sending coinbase-to-coinbase, otherwise it will take about 1 hour, 'complete' means it cannot be reversed or canceled
r.transaction.id
=> '501a1791f8182b2071000087'
r.transaction.recipient.email
=> 'user@example.com'
r.to_hash
=> ... # raw hash response
```

You can also send money in other [currencies](https://github.com/coinbase/coinbase-ruby/blob/master/supported_currencies.json).  It will be automatically converted to the correct BTC amount using the current exchange rate.

```ruby
r = coinbase.send_money 'user@example.com', 1.23.to_money('USD')
r.transaction.amount.format
=> "0.06713955 BTC"
```

The first parameter can also be a bitcoin address and the third parameter can be a note or description of the transaction.  Descriptions are only visible on Coinbase (not on the general bitcoin network).

```ruby
r = coinbase.send_money 'mpJKwdmJKYjiyfNo26eRp4j6qGwuUUnw9x', 2.23.to_money("USD"), "thanks for the coffee!"
r.transaction.recipient_address
=> "mpJKwdmJKYjiyfNo26eRp4j6qGwuUUnw9x"
```

### Request bitcoin

This will send an email to the recipient, requesting payment.

```ruby
r = coinbase.request_money 'client@example.com', 50, "contractor hours in January (website redesign for 50 BTC)"
r.transaction.request?
=> true
r.transaction.id
=> '501a3554f8182b2754000003'
r = coinbase.resend_request '501a3554f8182b2754000003'
r.success?
=> true
r = coinbase.cancel_request '501a3554f8182b2754000003'
r.success?
=> true
# from the other account
r = coinbase.complete_request '501a3554f8182b2754000003'
r.success?
=> true
```

### List your current transactions

Sorted in descending order by timestamp, 30 per page.  You can pass a pass an integer as the first param to page through results (for example `coinbase.transactions(2)`).

```ruby
r = coinbase.transactions
r.current_page
=> 1
r.num_pages
=> 7
r.transactions.collect{|t| t.transaction.id }
=> ["5018f833f8182b129c00002f", "5018f833f8182b129c00002e", ...]
r.transactions.collect{|t| t.transaction.amount.format }
=> ["-1.10000000 BTC", "42.73120000 BTC", ...]
```

### Buy or Sell bitcoin

Buying and selling bitcoin requires you to [link and verify a bank account](https://coinbase.com/payment_methods) through the web app first.

On a buy, we'll debit your bank account and the bitcoin will arrive in your Coinbase account four business days later (the `payout_date`, this is how long it takes for the bank transfer to complete, although we're working on shortening this window).

On a sell we'll credit your bank account and it will arrive within two business days.

```ruby
r = coinbase.buy! 1
r.success?
=> true
r.transfer.code
=> '6H7GYLXZ'
r.transfer.status
=> 'created'
t.total.format
=> "$17.95"
r.transfer.payout_date
=> "2013-02-01T18:00:00-08:00"
```


```ruby
r = coinbase.sell! 1
r.success?
=> true
r.transfer.code
=> 'RD2OC8AL'
t.total.format
=> "$17.93"
r.transfer.payout_date
=> "2013-02-01T18:00:00-08:00"
```

### Check bitcoin prices

Check the buy or sell price by passing a `quantity` of bitcoin that you'd like to buy or sell.  This price includes Coinbase's fee of 1% and the bank transfer fee of $0.15.

```ruby
coinbase.buy_price(1).format
=> "$17.95"
coinbase.buy_price(30).format
=> "$539.70"
```


```ruby
coinbase.sell_price(1).format
=> "$17.93"
coinbase.buy_price(30).format
=> "$534.60"
```

### Create a payment button

This will create the code for an embeddable payment button (and modal window) on your site to accept bitcoin.  You can read [more about payment buttons here and see a demo](https://coinbase.com/docs/merchant_tools/payment_buttons).

The arguments are (in order): item name, price, descripion, custom param (which comes through in the [callback](https://coinbase.com/docs/merchant_tools/callbacks) to your site).

```ruby
r = coinbase.create_button "Your Order #1234", 42.95.to_money('EUR'), "1 widget at €42.95", "1234"
r.code
=> "93865b9cae83706ae59220c013bc0afd"
r.embed_html
=> "<div class=\"coinbase-button\" data-code=\"93865b9cae83706ae59220c013bc0afd\"></div><script src=\"https://coinbase.com/assets/button.js\" type=\"text/javascript\"></script>"
```

### Create a new user

```ruby
r = coinbase.create_user "newuser@example.com"
r.user.email
=> "newuser@example.com"
r.receive_address
=> "mpJKwdmJKYjiyfNo26eRp4j6qGwuUUnw9x"
```

## Adding new methods

You can see a [list of method calls here](https://github.com/coinbase/coinbase-ruby/blob/master/lib/coinbase/client.rb) and how they are implemented.  They are a wrapper around the [basic JSON api](https://coinbase.com/api/doc).

If there are any methods listed in the [api reference](https://coinbase.com/api/doc) that don't have an explicit function name in the gem, you can also call `get`, `post`, `put`, or `delete` with a `path` and optional `params` hash to get back a basic JSON response.  For example:

```ruby
coinbase.get('/account/balance')
=> {"amount"=>"50.00000000", "currency"=>"BTC"}
```

Or feel free to add a new wrapper method and submit a pull request.

## Security Notes

If someone gains access to your API Key they will have complete control of your Coinbase account.  This includes the abillity to send all of your bitcoins elsewhere.

For this reason, API access is disabled on all Coinbase accounts by default.  If you decide to enable API key access you should take precautions to store your API securely in your application.  How to do this is application specific, but it's something you should [research](http://programmers.stackexchange.com/questions/65601/is-it-smart-to-store-application-keys-ids-etc-directly-inside-an-application) if you don't already have a solution in place.

## Testing

If you'd like to contribute code or modify this gem, you can run the test suite with:

```ruby
gem install coinbase --dev
bundle exec rspec # or just 'rspec' may work
```

## Contributing

1. Fork this repo and make changes in your own copy
2. Add a test if applicable and run the existing tests with `rspec` to make sure they pass
3. Bump the version number in `lib/coinbase/version.rb`
4. Add your name to the contributors list (or we can add you)
5. Commit your changes and push to your fork `git push origin master`
6. Create a new pull request and submit it back to us!
