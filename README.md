# Amazon Orders

A collection of Rake tasks for scraping _Amazon.com_ and fetching info about your order history. Useful for determining stats about your purchase habits or automatic fetching of open order status like delivery dates and tracking numbers. 

Data is stored in a SQLite database and ActiveRecord adapters are provided.

## Install

```bash
git clone https://github.com/kirPoNik/amazon-orders.git
cd amazon-orders
bundle
bundle exec rake orders:fetch
```

## Usage

Once you've imported your orders (it can take a while!), you can send orders to slack`. 

You'll get output that looks something like this:

```bash
$  rake get_latests

## Advanced Usage

The following Rake tasks are available:

| Task Name | Description |
| ------------- | ----------- |
| `rake stats` | Display some interesting stats about your order history.
| `rake orders:fetch` | Log in to Amazon.com and fetch all orders in your history.
| `rake orders:get_latests` | Select orders from DB and send to slack 
| `rake config:generate` | Generate a `config/account.yml` file with your Amazon.com credentials.
| `rake slack_config:generate` | Generate a `config/slack.yml` file with slack URL
| `rake db:reset` | Wipe all local order data.
| `rake db:migrate` | Ensure the DB schema is up-to-date.
| `rake console` | Open an interactive console with the environment loaded (helpful if you want run your own queries or poke around your data).

## License

[MIT License](https://github.com/chrisb/amazon-orders/blob/master/LICENSE). Copyright 2015 Chris Bielinski.
