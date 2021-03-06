# Rack::Attack!!!
*A DSL for blocking & thottling abusive clients*

Rack::Attack is a rack middleware to protect your web app from bad clients.
It allows *whitelisting*, *blacklisting*, and *thottling* based on arbitrary properties of the request.

Thottle state is stored in a configurable cache (e.g. `Rails.cache`), presumably backed by memcached.

## Installation

Install the [rack-attack](http://rubygems.org/gems/rack-attack) gem; or add it to you Gemfile with bundler:

    # In your Gemfile
    gem 'rack-attack'

Tell your app to use the Rack::Attack middleware.
For Rails 3 apps:

    # In config/application.rb
    config.middleware.use Rack::Attack

Or for Rackup files:

    # In config.ru
    use Rack::Attack

Optionally configure the cache store for throttling:

    Rack::Attack.cache.store = ActiveSupport::Cache::MemoryStore.new # defaults to Rails.cache

Note that `Rack::Attack.cache` is only used for throttling; not blacklisting & whitelisting. Your cache store must implement `increment` and `write` like [ActiveSupport::Cache::Store](http://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html).

## How it works

The Rack::Attack middleware compares each request against *whitelists*, *blacklists*, and *throttles* that you define. There are none by default.

 * If the request matches any whitelist, it is allowed. Blacklists and throttles are not checked.
 * If the request matches any blacklist, it is blocked. Throttles are not checked.
 * If the request matches any throttle, a counter is incremented in the Rack::Attack.cache. If the throttle limit is exceeded, the request is blocked and further throttles are not checked.

## Usage

Define blacklists, throttles, and whitelists as blocks that return truthy of falsy values.
A [Rack::Request](http://rack.rubyforge.org/doc/classes/Rack/Request.html) object is passed to the block (named 'req' in the examples).

### Blacklists

    # Block requests from 1.2.3.4
    Rack::Attack.blacklist('block 1.2.3.4') do |req|
      # Request are blocked if the return value is truthy
      '1.2.3.4' == req.ip
    end

    # Block logins from a bad user agent
    Rack::Attack.blacklist('block bad UA logins') do |req|
      req.path == '/login' && req.post? && req.user_agent == 'BadUA'
    end

### Throttles

    # Throttle requests to 5 requests per second per ip
    Rack::Attack.throttle('req/ip', :limit => 5, :period => 1.second) do |req|
      # If the return value is truthy, the cache key for the return value
      # is incremented and compared with the limit. In this case:
      #   "rack::attack:#{Time.now.to_i/1.second}:req/ip:#{req.ip}"
      #
      # If falsy, the cache key is neither incremented nor checked.

      req.ip
    end

    # Throttle login attempts for a given email parameter to 6 reqs/minute
    Rack::Attack.throttle('logins/email', :limit => 6, :period => 60.seconds) do |req|
      request.path == '/login' && req.post? && req.params['email']
    end

### Whitelists

    # Always allow requests from localhost
    # (blacklist & throttles are skipped)
    Rack::Attack.whitelist('allow from localhost') do |req|
      # Requests are allowed if the return value is truthy
      '127.0.0.1' == req.ip
    end

## Responses

Customize the response of blacklisted and throttled requests using an object that adheres to the [Rack app interface](http://rack.rubyforge.org/doc/SPEC.html).

    Rack:Attack.blacklisted_response = lambda do |env|
      [ 503, {}, ['Blocked']]
    end

    Rack:Attack.throttled_response = lambda do |env|
      # name and other data about the matched throttle
      body = [
        env['rack.attack.matched'],
        env['rack.attack.match_type'],
        env['rack.attack.match_data']
      ].inspect

      [ 503, {}, [body]]
    end

For responses that did not exceed a throttle limit, Rack::Attack annotates the env with match data:

    request.env['rack.attack.throttle_data'][name] # => { :count => n, :period => p, :limit => l }

## Logging & Instrumentation

Rack::Attack uses the [ActiveSupport::Notifications](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html) API if available.

You can subscribe to 'rack.attack' events and log it, graph it, etc:

    ActiveSupport::Notifications.subscribe('rack.attack') do |name, start, finish, request_id, req|
      puts req.inspect
    end

## Motivation

Abusive clients range from malicious login crackers to naively-written scrapers.
They hinder the security, performance, & availability of web applications.

It is impractical if not impossible to block abusive clients completely.

Rack::Attack aims to let developers quickly mitigate abusive requests and rely
less on short-term, one-off hacks to block a particular attack.

Rack::Attack complements tools like iptables and nginx's [limit_zone module](http://wiki.nginx.org/HttpLimitZoneModule).

[![Travis CI](https://secure.travis-ci.org/ktheory/rack-attack.png)](http://travis-ci.org/ktheory/rack-attack)

## License

Copyright (c) 2012 Kickstarter, Inc

Released under an [MIT License](http://opensource.org/licenses/MIT)
