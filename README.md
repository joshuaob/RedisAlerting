# RedisAlerting

An alerting engine that uses keys from redis to determine if a reading is out of range.  It then writes to a key in redis to indicate that the reading is out of range.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'redis_alerting'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install redis_alerting

## Usage

To start and daemonize the engine:

```sh
redis-alerting start -- -c path/to/config.yml
```

To run it with logging output:

```sh
redis-alerting run -- -c path/to/config.yml -l info
#redis-alerting run -- -c path/to/config.yml -l debug # this will be noisy!
```

Check the status:

```sh
redis-alerting status
```

Stop it:

```sh
redis-alerting stop
```

## How it works

The gem uses specific key patterns to get the min and the max limits for a reading.

Given the config file below we will describe how the gem would check some limits and write back to redis to inidicate an out of range state.

```yaml
---
:interval: 1                           # how often to check the readings
:namespace: alerts                     # the namespace to use for tracking active alerts/conditions
:separator: .                          # redis namespace separator
:channel: alerts                       # publish alert messages to this channel
:faye_url: http://localhost/faye       # the URL of the Faye endpoint to publish to
:extrema:                              # details for where to find extrema information
  :min: min                            # what you call your minimum
  :max: max                            # what you call your maximum
  :pattern: settings.$source_$extrema  # the pattern to locate your extrema keys
:sources:                              # keys to obtain values from that need to be checked
  :ph: readings.ph
  :ec: readings.ec
  :flow_rate: readings.flow_rate
# :name: redis.key.with.live.value
```

### Extrema Patterns

So to find the min/max keys you will need to specify the pattern your extrema keys use (you use a pattern right?)

In the above case `settings.$source_$extrema` would be translated into:

* `settings.ph_min`
* `settings.ph_max`
* `settings.flow_rate_min`
* `settings.flow_rate_max`

If you stored your extrema a bit differently, a pattern like this might suffice: `alerts:$source:$extrema`:

* `alerts:ph:min`
* `alerts:ph:max`
* `alerts:ec:min`
* `alerts:ec:max`

Get it?

### Sources

With the example of `ph` above, the alerting system would check the redis key `readings.ph` and determine if it was outside the limits set in `alerts.ph.max` and `alerts.ph.min`.

When it finds that the value in `readings.ph` is outside the range, it will add "ph" to the redis set `alerts.active`.  When it comes back into range "ph" will be removed from the `alerts.active` set.

So to quickly summarize:

* `readings.ph` - the reading value used to check against the min and max settings
* `settings.ph_min` - the minimum value for the reading (below which an alert is raised)
* `settings.ph_max` - the maximum value for the reading (above which an alert is raised)

### Active Set

At anytime you can query the contents of the `alerts.active` SET to see which alert sources are currently active.  It will contain the name specified under the sources section.

### Published messages

When an alert condition is added or removed, the following message will be published the channel specified in the config file in JSON format:

So when an alert is raised, this message will be published:

```json
{
  "action"   : "add",
  "name"     : "ec",
  "condition": "high",
  "value"    : 6.2,
  "min"      : 0.1,
  "max"      : 5.8
}
```

When an alert is no longer active, this message will be published:

```json
{
  "action" : "remove",
  "name"   : "flow",
  "value"  : 2.4,
  "min"    : 0.1,
  "max"    : 5.8
}
```

#### Publishing to Faye

There is also support for publishing to a [Faye](http://faye.jcoglan.com/) URL.  Simply put the Faye URL in your config file like so:

```yaml
:faye_url: http://localhost/faye
:channel: alerts
```

Or set it at command line:

    $ redis-alerting run -- -c config.yml --faye-url http://localhost/faye

This will disable publishing to redis.  The channel will automatically be set to `/alerts` if you leave out the leading slash.

### Simple example

With the engine started try this (still sticking with the keys in the config file above):

```sh
$ redis-cli
127.0.0.1:6379> set settings.ph_min 4000
127.0.0.1:6379> set settings.ph_max 9000
127.0.0.1:6379> set readings.ph 6000
127.0.0.1:6379> smembers alerts.active
(empty list or set)
127.0.0.1:6379> set readings.ph 9100
127.0.0.1:6379> smembers alerts.active
1) "ph"
127.0.0.1:6379> set readings.ph 3900
127.0.0.1:6379> smembers alerts.active
1) "ph"
127.0.0.1:6379> set readings.ph 6000
127.0.0.1:6379> smembers alerts.active
(empty list or set)
```

## Todo

* more specs

## Contributing

1. Fork it ( https://github.com/AutogrowSystems/RedisAlerting/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
