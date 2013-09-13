## Bucky Server

Bucky uses a Node server to forward the HTTP requests with your monitoring data to
Statsd/Graphite, OpenTSDB, etc.

## Hosting

Everything you need to run Bucky on Heroku or Nodejitsu is included, just update the
[config/defaults.yaml](config file) and push to the service of your choice.

### Heroku

```bash
heroku create
git push heroku master
```

### Nodejitsu

```bash
# In BuckyServer/

jitsu deploy
```

### EC2 / Bare Metal

If you'd rather host Bucky on EC2 directly or on raw hardware, you just need something
which will run `./start.js` in a reliable way.

You can use environment variables to control runtime options, or put them in your config
file in the `Server` section.

```bash
# In BuckyServer/

npm install
PORT=3333 APP_ROOT=bucky/ ./start.js
```

The `APP_ROOT` (or `config.Server.AppRoot`) will prefix all endpoints.

Bucky will respond to all requests at `/APP_ROOT/warmup`, if you need a health check url.

Bucky can be setup to receive data at multiple endpoints, but by default it listens
to `/APP_ROOT/send` on whichever port you specify.

## Configuring

The Bucky server is designed to be as modular as is possible to allow you
to customize it to your needs without having to modify the core app.

We use 'modules' (which are just node modules) to break out portions of the
app.  You can specify which modules you'd like to use in the `config/default.yaml`
config file.

### Modules

- Logger
  Use to have Bucky log to something other than the console
- Config
  Use to have Bucky pull config from somewhere other than the default file
- App
  Use to do things when Bucky loads and/or on requests.  Auth, monitoring initialization, etc.
- Collectors
  Use to send Bucky data to new and exciting places.

We can only have one logger and one config, but you can specify as many app and collector modules
as you like.

All modules follow the same basic sketch.  You export a method which is called when Bucky
starts up.  That method is provided with as much of `{app, config, logger}` as we have
loaded so far, and is expected to call a callback when it's ready for the loading to continue.

#### Logger

Used to log output.  Defaults to a wrapper around console.log/console.error.

Should export a function which will be called when the server is started:

```coffeescript
module.exports = ({logger}, next) ->
  # logger is the previous logger (just the console)

  next myNewLogger
```

This function should call the callback with a logger object which implements
`log` and `error`:

```coffeescript
module.exports = ({logger}, next) ->
  myNewLogger = {
    log: ->
      console.log "Bucky message:", arguments...
    error: ->
      console.error "Bucky error:", arguments...
  }

  next myNewLogger
```

#### Config

By default config comes from the config files loaded using the node `config` module.

If specified, this module will replace that config.  Please note that the list of
modules comes from the `config` module, so the list of modules must always be
specified there.  All other config options can be moved to your config solution
of choice using this extension point.

At HubSpot, we use a config solution which supports live reloading of config
values.  For this reason, the config api is a bit more complex than you might
expect.  A wrapper is provided in `lib/configWrapper.coffee` for you to use
should you wish to use a simpler solution.

```coffeescript
module.exports = ({config, logger}, next) ->
  # config is the old config which was being used
  # before this module was loaded

  # logger.log and logger.error should be used rather than
  # console

  next myConfigObject
```

A config value will be retrieved from the config object the callback is
called with like this:

```coffeescript
  config.get('some.config').get()

  config.get('some.config').on 'change', ->
    # The config changed!
```

You are free to implement the `on` method as a dud if live reloading doesn't
make sense using your config system.  Take a look at [lib/configWrapper.coffee](lib/configWrapper.coffee)
for an example of how a basic object can be converted.
  
#### App

App modules get loaded once, and can optionally provide a function to be ran with each request.

Simple app modules are a good place to put any server config, initialization code, etc.

We use app modules to make [modules/trustProxy.coffee](little tweaks) to how express works and
[modules/sentry.coffee](enable monitoring).

App modules are called at initialize-time with a hash including a reference to the express app:

```coffeescript
module.exports = ({app, logger, config}, next) ->
```

If your app module calls the callback with a function, that function will be executed on all requests to
`/send`, which is the default endpoint.

If the callback is called with a hash, it is expected to be a mapping between endpoints and handler functions.

```coffeescript
module.exports = ({app, logger, config}, next) ->
  next
    send: (req, res) ->
      # Standard express request handling stuff in here

    someOtherEndpoint: (req, res) ->
       # Will get requests which are sent to /someOtherEndpoint
```

These functions work like middleware, they are called sequentially.  You can use them to implement
things like [modules/auth.coffee](auth) if you need it.

#### Collectors

It's not a standard type of module (the core of Bucky has no idea about it), but the default
[modules/collectors.coffee](collectors app module) looks to a forth type of module to know
where to send data.

[modules/statsd.coffee](Statsd) and [modules/openTSDB.coffee](OpenTSDB) collectors are included.

Collectors should export a function which is called on initialize, and call the callback with a hash
mapping endpoints to handlers.

```coffeescript
module.exports = ({app, logger, config}, next) ->
  next
    send: (data) ->
      # This collector will receive any requests to /send (the default endpoint)

      logger.log "We got some data!"
```