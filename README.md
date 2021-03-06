## Bucky Server

Bucky uses a Node server to forward the HTTP requests with your monitoring data to
Statsd/Graphite, OpenTSDB, or whatever other service you'd like.

Also see the [Bucky Client](http://github.hubspot.com/BuckyClient/).

## Hosting

Everything you need to run Bucky on Heroku or Nodejitsu is included, just update the
[config file](config/default.yaml) and push to the service of your choice.

### Heroku

```bash
heroku create
git push heroku master
```

### Nodejitsu

```bash
jitsu deploy
```

The jitsu application will ask you for a subdomain to run the service on, and will
increment the version of the application whenever you deploy.

### EC2 / Bare Metal

If you'd rather host Bucky on EC2 directly or on raw hardware, you just need something
which will run `./start.js` in a reliable way.

You can use environment variables to control runtime options, or put them in your [config file](config/default.yaml)
in the `server` section.

You'll need to have [nodejs](http://nodejs.org/) installed.  Anything in the 0.8.x series or above should work
fine.  We recommend using [nvm](https://github.com/creationix/nvm), as it gives you an extra dimention
of flexability, but using your system's package manager should work just as well.

```bash
# In the project directory:

npm install
PORT=3333 APP_ROOT=bucky/ ./start.js
```

The `APP_ROOT` (or `config.server.appRoot`) will prefix all endpoints.

Bucky will respond to all requests at `/APP_ROOT/v1/health-check`, if you need a health check url.

Bucky can be setup to receive data at multiple endpoints, but by default it listens
to `/APP_ROOT/v1/send` on whichever port you specify.

#### Ubuntu (12.04)

```bash
# Install nodejs
# This assumes you're on a 64 bit machine
wget http://nodejs.org/dist/v0.10.19/node-v0.10.19-linux-x64.tar.gz
tar xvf node-v0.10.19-linux-x64.tar.gz 
sudo ln -s `pwd`/node-v0.10.19-linux-x64/bin/{node,npm} /usr/local/bin/

# Grab a Bucky release
# You should use the latest release available at https://github.com/HubSpot/BuckyServer/releases
wget https://github.com/HubSpot/BuckyServer/archive/v0.2.0.tar.gz -O BuckyServer.tar.gz
tar xvf BuckyServer.tar.gz
cd BuckyServer

# Install Bucky
sudo npm install -g

# Make any config changes by editing /usr/local/lib/node_modules/bucky-server/config/default.yaml

# You can start bucky by running bucky-server

# Add the upstart script so Bucky starts on startup and respawns
sudo cp init/bucky-server.conf /etc/init/

# Start Bucky with
sudo start bucky-server

# Log files will appear in /var/log/bucky.log by default
```

You can run bucky on a specific port (make sure to open that port in your security group
if you're using EC2), or you can use a reverse proxy like Nginx or HAProxy to serve it
on the same domain and port as your website, it's up to you.

## Configuring

If you're not already running a stats collection service, you should take a look at our
[help doc](http://github.hubspot.com/BuckyServer/getting_started/choosing_a_stats_service/).

Most people will only need to specify [the config](config/default.yaml) they're interested in
and start up the server.

If you need more customization, you can write a module:

### Modules

There are a few of types of modules:


- Logger - Use to have Bucky log to something other than the console
- Config - Use to have Bucky pull config from somewhere other than the default file
- App - Use to do things when Bucky loads and/or on requests.  Auth, monitoring initialization, etc.
- Collectors - Use to send Bucky data to new and exciting places.

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

At HubSpot, we're believers in config which can be changed without restarting
services. For this reason, the config api is a bit more complex than you might
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
for an example of how a basic object can be converted (and feel free to use it).
  
#### App

App modules get loaded once, and can optionally provide a function to be ran with each request.

Simple app modules are a good place to put any server config, initialization code, etc.

We use app modules to make [little tweaks](modules/trustProxy.coffee) to how express works and enable
monitoring.

App modules are called at initialize-time with a hash including a reference to the express app:

```coffeescript
module.exports = ({app, logger, config}, next) ->
```

If your app module calls the callback with a function, that function will be executed on all requests to
`/v1/send`, which is the default endpoint.

If the callback is called with a hash, it is expected to be a mapping between endpoints and handler functions.

```coffeescript
module.exports = ({app, logger, config}, next) ->
  next
    send: (req, res, _next) ->
      # Standard express request handling stuff in here

    someOtherEndpoint: (req, res, _next) ->
       # Will get requests which are sent to /v1/someOtherEndpoint
```

These functions work like middleware, they are called sequentially.  You can use them to implement
things like [auth](modules/auth.coffee) if you need it.

#### Collectors

It's not a standard type of module (the core of Bucky has no idea about it), but the default
[collectors app module](modules/collectors.coffee) looks to a fourth type of module to know
where to send data.

[Statsd](modules/statsd.coffee) and [OpenTSDB](modules/openTSDB.coffee) collectors are included.

Collectors should export a function which is called on initialize, and call the callback with a hash
mapping endpoints to handlers.

```coffeescript
module.exports = ({app, logger, config}, next) ->
  next
    send: (data) ->
      # This collector will receive any requests to /v1/send (the default endpoint)

      logger.log "We got some data!"
```

### Format

If you are interested in writing new clients, the format of metric data is the same as is used by statsd:

```
<metric name>:<metric value>|<unit>[@<sample rate>]
```

For example:

```
my.awesome.metric:35|ms
some.other.metric:3|c@0.5
```

All requests are sent with content-type `text/plain`.
