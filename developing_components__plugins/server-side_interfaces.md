# Server-side interfaces

<!-- toc -->

## Server-side Providers

### ui
Package: [`hive-ui`](https://github.com/hivejs/hive-ui)

#### ui.registerModule(file:String)
register a file that must adhere to the component definition spec and will be loaded on the client-side.

#### ui.registerJavascript(file:String)
register a file that will be processed by browserify as an entry file (meaning it will be run when the bundle is loaded as if it was included separately with a script tag). The advantage is that you can utilize browserify's magic and `require` your files like normal. If your module is very large and not always needed, you should use `ui.externalizeModule` (for modules that use `module.exports`) or include it manually by registering a static dir (see below) to expose the file. In both cases you can make sure to load the file on the client only once by using `ui.requireScript`.

#### ui.externalizeModule(moduleName:String)
puts the given module in a separate bundle, available in `/statis/build/<moduleName>.js`. After loading that bundle with `ui.requireScript` on the client, you can require the module as usual. Magic!

#### ui.registerExternalModule(moduleName:String)
exclude a module from the main javascript bundle (or rather declare it to be external). If you want to be able to require it, bundle it yourself with `browserify#require`, expose it on the server by registering a static dir and load it on the client with `ui.requireScript`.

Usually you can use `externalizeModule`, which does all that for you, but sometimes you have only a pre-built bundle available -- that's when this method comes in handy.

#### ui.registerStaticDir(dir:String, [options])
register `dir`, a sub-directory of your component, which will be available at `/static/hive-component-name/path/to/dir`. The `options` will be passed to koa-static-cache.

#### ui.bundle*()
returns the bundled client-side javascript.

#### ui.bootstrapMiddleware()
returns koa middleware which sends the bootstrapping html of the web ui

#### ui.getBootstrapCode()
returns the bootstrapping html of the web ui.

### auth
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### auth.registerAuthenticationProvider(type:String, provider:Generator)
register an authentication provider for an authentication type which should have the following signature: `function*(credentials)` and should return a user object or, if authentication failed, a falsy value.

#### auth.registerAuthorizationProvider(provider:GeneratorFunction)
register an authorization provider which should have the following signature: `function*(user, action, data)`, where `user` is a user object, `action` is a string that identifies the action (e.g. `"document:write"`), and `data` is an object providing additional information (e.g. `{document: <documentId>}`), and should return either a boolean indicating authorization granted/denied or `null` to indicate a neutral vote, leaving the decision up to other registered providers. Authorization providers should also take into account an optionally defined `scopes` field on the `user` object, which defines scopes that an authorized app posses to operate on the user's behalf.

#### auth.authenticate*(authType:String, credentials:mixed)
tries to authenticate a user using the provider registered for `authType` and the provided `credentials`. Returns a user object or a falsy value.

#### auth.authorize*(user:Object, action:String, data:Object)
tries to authorize an `action` committed by a `user`, where `user` is a user object, `action` is a string that identifies the action (e.g. `"document:write"`), and `data` is an object providing additional information (e.g. `{document: <documentId>}`). Returns `true` (authorization granted) if at least one authorization provider voted `true` while all others voted `null`, and it returns `false` (authorization denied) if at least one authorization provider voted `false` or all voted `null`.

### authToken
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### authToken.sign*(payload:Object)
returns a jsonwebtoken from `payload` with the secret set in the config file, where `payload` should be an object containing a user `{user: <userId>}`, and optionally some scopes. `hive-auth-token` also registers a matching `token` authentication provider (see [hive-auth-token]()).

### importexport
Package: `hive-core`

#### importexport.registerExportProvider(type:string, exportType:string, provider:GeneratorFunction(document, snapshot):Buffer|string)
* `type` is the ot type of the document as registered with the ot provider
* `exportType` is the mime type of the exported file
The registered export provider takes the `document`and `snapshot` objects and returns the exported file.

#### importexport.export*(snapshotId:string, exportType:string):Buffer|string
Checks if an export provider for the given exportType exists and calls it, returning the file contents.

#### importexport.registerImportProvider(type:string, importType:string, provider:GeneratorFunction(document, user, data))
* `type` is the ot type of the document as registered with the ot provider
* `importType` is the mime type of the file that is being imported
The registered import provider takes the `document` object and the `user` object (of the user on whose behalf the import is carried out) and imports the file contents passed as `data`.

#### importexport.import*(documentId:Number, user:Object, importType:string, data:Buffer|string)
Checks if an import handler for the given import type is registered andd calls it.

### broadcast
Package: [`hive-core`](https://github.com/hivejs/hive-core)

There's two types of broadcasts (which are multiplexed over the transport stream): the normal per-document broadcasts and the sync broadcasts, over which workers publicise newly committed edits.

#### broadcast.registerTransport(stream:DuplexStream)
registers a broadcast transport. `stream` should be a duplex stream that will send all data written to it to the other workers and will emit such data sent by other workers. This is implemented by `hive-broadcast-smokesignal` and `hive-broadcast-memory` (which is technically a noop, as it is only to be used for single-worker setups).

#### broadcast.registerChannel(id:Buffer, fn:Function)
registers a broadcast channel with `id` along with a setup function `fn` which will be called for every client with `(user, document, client, brdcst)`, where `user` is the user object corresponding to the user connecting, `document` is the id of the document the user is on, `client` is a duplex stream representing the client (i.e. if you `write()` to it, the client will receive it and everything the client writes to it, you can `read()` here) and `brdcst` is a duplex stream representing all other clients (i.e. if you `write()` to it, all other setup instances will receive it on the same stream, and everything they write, you can `read()` here). So, as you might have guessed already, you will usually want to pipe the two streams together in order to achieve a broadcast experience. You may however use the setup function to put in place validation and auth middleware, which can be conveniently added as `through` streams.

#### broadcast.document(document:Number, user:Object)
Returns a duplex stream which acts as a broadcast proxy for the specified `document` tailored to the specified `user`. Interface implementations should use this to pass on access to the broadcast.

#### broadcast.sync(document:Number)
Returns a duplex stream that allows yo to broadcast newly committed edits to and receive such from other workers. Used by `hive-sync`.

### cli
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### cli.registerCommand(cmd:String, fn:Function)
Registers a command `fn` with the name `cmd`. This won't throw if the command has already been registered and will instead override the earlier registration!

#### cli.command(cmd:String, argv:[String])
Executes a registered command.

#### cli.dispatch(argv:[String])
Inspects the command line arguments passed via `argv` and calls the corresponding subcommand with the *whole* `argv`, e.g. if you run `hive test --foo` it will call command `"test"` with `argv` set to `["node",".../hive","test","--foo"]`.

The main command is the empty string `""` and is reserved for hive-services, but you can override that by registering a new function for the empty command `""`.

### services
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### services.registerService(name:String, fn:Function)
registers a function `fn` as a service called `name`.

#### services.startService(name:String, opts:Object, [runInternally:Boolean])
starts a service called `name` with some options defined in `opts`. `runInternally` determines whether the service will be executed in the current process or inf a new process. By default services are started externally.

### config
Package: [`hive-core`](https://github.com/hivejs/hive-core)

The config provider provides

1. the contents of `default.json` in the config dir of your instance
2. with higher priority: the contents of the config file correspnding to the `NODE_ENV` env var.
3. Even higher priority have env vars which follow the form `hive_foo_bar` (for setting config key `foo:bar`).

#### config.get(key)
The config is a big object. Components and plugins should put their config under a key that is prefixed with the component's name. The separator for keys is the colon `:`, e.g. if your config file looks like this

```json
{
  "test": {
    "timeout": 10000
  }
}
```

you'd get that value with `config.get('test:timeout')`, however it's also possible to get the whole object, i.e. `config.get('test')` will return `{timeout:10000}`.

### hooks
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### hooks.on(hook:String, fn:Generator)
#### hooks.registerHook(hook:String, fn:Generator)
registers a hook called `hook` with a generator function that will be called with the hook arguments.

#### hooks.callHook*(hook:String,[args...])
calls hooks registered for the provided name `hook` with `args`.

### http
Package: [`hive-core`](https://github.com/hivejs/hive-core)

This is a koa instance.

#### http.use([path:String], middleware:Generator)
The router middleware is added right before the http server is started, so any middleware you add via `http.use()` will execute before the router.

#### http.router
This is a router as implemented by koa-router.

#### http.router.VERB(path:String, fn:Generator)

#### http.server
The http server instance that will be bound to the port and will call the koa instance.

### logger
Package: [`hive-core`](https://github.com/hivejs/hive-core)

This is a log4js instance.

#### logger.getLogger(name:String)
returns a logger instance that features the following methods:

 * `trace`
 * `debug`
 * `info`
 * `warn`
 * `error`
 * `fatal`

### ot
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### ot.registerOTType(name:String, ottype:Object)
registers an OT type that adheres to the shareJS ottype spec.

#### ot.getOTType(name:String)
returns an OT type registered under `name`.

### queue
Package: [`hive-core`](https://github.com/hivejs/hive-core)

This provider should coordinate with other workers to ensure that queue submissions from all workers are processed and there's never two workers running tasks of the same queue in parallel. The default implementation is `hive-queue` which registers a service `queue` which does the coordination.

#### queue.document(document:Number)
returns a queue for the passed `document` with the following interface:
```javascipt
q.push(function(cb){
  // ... do something ...
})
```

### sync
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### sync.getDocument*(document:Number)
returns a gulf document fully wired with queue and adapter. Use this to interface with the gulf document mechanics. And have a look at [the gulf]() docs to see how it works.

#### sync.createDocument*(type)
creates a document with default contents according to the ottype you specify. Use `sync.getDocument*()` to obtain the corresponding gulf document.

### oauth
Package: [`hive-core`](https://github.com/hivejs/hive-core)

#### oauth.registerClientAuthenticator(authenticator:Generator)
Takes a generator function that will be called with the following arguments to authenticate an OAuth client: `(client_id, client_secret, redirect_uri)`. You should check if either `client_secret` or `redirect_uri` is `null` and shouldn't compare them to the stored values in that case. You should, however, throw an error if one of the arguments is `undefined`! You should return `true` if the values match, and `false` if not.

#### oauth.authenticateClient*(client_id, client_secret, redirect_uri)
Uses the previously registered client authenticator to authenticate the passed client. Returns a boolean indicating whether the values match, aka the client is authenticated.

## Server-side hooks

### orm:initialize (settings:Object)
 * `settings` the waterline settings object

Called before initializing waterline. You may want to use this to load your own collections with `waterline.loadCollection()`. It may however be more convenient to use `models:load`.

### orm:initialized (models:Object)
 * `models` an object featuring `user`, `document` and `snapshot` collections (in the default configuration). These are full-fledged waterline collections with all [CRUD methods as per the waterline docs]()

### models:load (models:Object)
 * `models` an object featuring the configurations for `user`, `document` and `snapshot` models

This is called before the models are created with `Waterline.Collection.extend` and loaded with `waterline#loadCollection`, so it is convenient for you to add your own attributes and models here.

### http:bindMiddleware
This hook is called when the http service is started, before the koa request handler is compiled and after the standard middleware has been added (before the router middleware, though, otherwise it would be pointless). Usually you will want to wait for this hook until you register your koa middleware. In rare cases, when you want to ensure your middleware is run first you can register your middleware right when your hive component is loaded.

### http:listening (server:HttpServer)
 * `server` the http server object

This hook is called once the server is listening. It is used for example in `hive-interface-stream` to install the shoe server.

### interface-stream:connect (plex:Stream)
 * `plex` A dataplex stream that allows you to add your own communication channel.

When this hook is called, the user is already authenticated.
It is not recommended to use this method unless you know what you are doing. Use broadcast channels instead.
