# Migration guide:

To get started upgrading your existing Sails app to version 1.0, follow the checklist below, which covers the changes most likely to affect the majority of apps.  If your app still has errors or warnings on startup after following the checklist, come back to this document and follow the applicable guides to upgrading various app components.

## tl;dr checklist -- things you simply _must_ do when upgrading to version 1.0

* **Install the `sails-hook-orm` module** into your app with `npm install --save sails-hook-orm`, unless your app has the ORM hook disabled.
* **Install the `sails-hook-sockets` module** into your app with `npm install --save sails-hook-sockets`, unless your app has the sockets hook disabled.
* **Install the `sails-hook-grunt` module** into your app with `npm install --save sails-hook-grunt`, unless your app has the Grunt hook disabled.
* **update your `config/globals.js` file** (if your app doesn't have `sails.config.globals` set to `false`) so that `models` and `sails` have boolean values (`true` or `false`), and `async` and `lodash` either have `require('async')` and `require('lodash')` respectively, or else `false`.  You may need to `npm install --save lodash` and `npm install --save async` as well.
* **The `/csrfToken` route** is no longer provided to all apps by default when using CSRF.  If you're utilizing this route in your app, add it manually to `config/routes.js` as `'GET /csrfToken': { action: 'security/grantCsrfToken' }`.
* **If your app uses CoffeeScript or TypeScript** see the [CoffeeScript](http://sailsjs.com/documentation/tutorials/using-coffee-script) and [TypeScript](http://sailsjs.com/documentation/tutorials/using-type-script) tutorials for info on how to update it.
* **If your app uses a view engine other than EJS**, you&rsquo;ll need to configure it yourself in the `config/views.js` file, and will likely need to run `npm install --save consolidate` for your project.  See the "Views" section below for more details.
* **If your app relies on views for the `badRequest` or `forbidden` responses**, you&rsquo;ll need add your own custom `api/responses/badRequest.js` or `api/responses/forbidden.js` files.  Those default responses no longer use views.

## Breaking changes to lesser-used functionality

* **Many resourceful pubsub methods have changed** (see the PubSub section below for the full list).  If your app only uses the automatic RPS functionality provided by blueprints (and doesn&rsquo;t call RPS methods directly), no updates are required.
* **Custom blueprints and the associated blueprint route syntax have been removed**.  This functionality can be replicated using custom actions, helpers and routes.  See the "Replacing custom blueprints" section below for more info.
* **Blueprint action routes no longer include `/:id?`** at the end -- that is, if you have a `UserController.js` with a `tickle` action, you will no longer get a `/user/tickle/:id?` route (instead, it will be just `/user/tickle`).  Apps relying on those routes should add them manually to their `config/routes.js` file.
* **`sails.getBaseUrl`**, deprecated in v0.12.x, has been removed.  See the [v0.12 docs for `getBaseUrl`](http://0.12.sailsjs.com/documentation/reference/application/sails-get-base-url) for more info and why it was removed and how you should replace it.
* **`req.params.all()`**, deprecated in v0.12.x, has been removed.  Use `req.allParams()` instead.
* **`sails.config.dontFlattenConfig`**, deprecated in v0.12.x, has been removed.  See the [original notes about `dontFlattenConfig`](http://sailsjs.com/version-notes/0point10-to-0point11-migration-guide#?config-files-in-subfolders) for details.
* **The order of precedence for `req.param()` and `req.allParams()` has changed.**  It is now consistently path > body > query (that is, url path params override request body params, which override query string params).
* **`req.validate()`** has been removed.  Use [`actions2`](http://sailsjs.com/documentation/concepts/actions-and-controllers#?actions-2) instead.
* **The default `res.created()` response has been removed.**  If you&rsquo;re calling `res.created()` directly in your app, and you don't have an `api/responses/created.js` file, you&rsquo;ll need to create one.  On a related note, the [Blueprint create action](http://sailsjs.com/documentation/reference/blueprint-api/create) will now return a 200 status code upon success, instead of 201.
* **The default `notFound` and `serverError` responses no longer accept a `pathToView` argument.** They will only attempt to serve a `404` or `500` view.  If you need to be able to call these responses with different views, you can customize the responses by adding `api/responses/notFound.js` or `api/responses/serverError.js` files to your app.
* **The <a href="https://www.npmjs.com/package/connect-flash" target="_blank">`connect-flash`</a> middleware has been removed** (so `req.flash()` will no longer be available by default).  If you wish to continue using `req.flash()`, run `npm install --save connect-flash` in your app folder and [add the middleware manually](http://sailsjs.com/documentation/concepts/middleware).
* **The `POST /:model/:id` blueprint RESTful route has been removed.**  If your app is relying on this route, you&rsquo;ll need to add it manually to `config/routes.js` and bind it to a custom action.
* **The `handleBodyParserError` middleware has been removed** -- in its place, the <a href="https://www.npmjs.com/package/skipper" target="_blank">Skipper body parser</a> now has its own `onBodyParserError` method.  If you have customized the [middleware order](http://sailsjs.com/documentation/concepts/middleware#?adding-or-overriding-http-middleware), you&rsquo;ll need to remove `handleBodyParserError` from the array.  If you've overridden `handleBodyParserError`, you&rsquo;ll need to instead override `bodyParser` with your own customized version of Skipper, including your error-handling logic in the `onBodyParserError` option.
* **The `methodOverride` middleware has been removed** -- if your app utilizes this middleware, simply `npm install --save method-override`, make sure your `sails.config.http.middleware.order` array (in `config/http.js`) includes `methodOverride` somewhere before `router` and add `methodOverride: require('method-override')()` to `sails.config.http.middleware`.

## Security

New apps created with Sails 1.0 will contain a **config/security.js** file instead of individual **config/cors.js** and **config/csrf.js** files, but apps migrating from earlier versions can keep their existing files as long as they perform the following upgrades:

* Change `module.exports.cors` to `module.exports.security.cors` in `config/cors.js`
* Change CORS config settings names to match the newly documented names in http://sailsjs.com/documentation/reference/configuration/sails-config-security-cors
* Change `module.exports.csrf` to `module.exports.security.csrf` in `config/csrf.js`
* `sails.config.csrf.routesDisabled` is no longer supported -- instead, add `csrf: false` to any route in `config/routes.js` that you wish to be unprotected by CSRF, for example:

```
'POST /some-thing': { action: 'do-a-thing', csrf: false },
```

## Views

For maximum flexibility, Consolidate is no longer bundled within Sails.  If you are using a view engine besides EJS, you'll probably want to install Consolidate as a direct dependency of your app.  Then you can configure the view engine in `config/views.js` like so:

```javascript
'extension': 'swig',
'getRenderFn': function() {
  // Import `consolidate`.
  var cons = require('consolidate');
  // Return the rendering function for Swig.
  return cons.swig;
}
```

Adding custom configuration to your view engine is a lot easier in Sails 1.0:

```javascript
'extension': 'swig',
'getRenderFn': function() {
  // Import `consolidate`.
  var cons = require('consolidate');
  // Import `swig`.
  var swig = require('swig');
  // Configure `swig`.
  swig.setDefaults({tagControls: ['{?', '?}']});
  // Set the module that Consolidate uses for Swig.
  cons.requires.swig = swig;
  // Return the rendering function for Swig.
  return cons.swig;
}
```

## PubSub

* Removed deprecated `backwardsCompatibilityFor0.9SocketClients` setting.
* Removed deprecated `.subscribers()` method.
* Removed deprecated "firehose" functionality.
* Removed support for 0.9.x socket client API.
* The following resourceful pubsub methods have been removed:
  * `.publishAdd()`
  * `.publishCreate()`
  * `.publishDestroy()`
  * `.publishRemove()`
  * `.publishUpdate()`
  * `.watch()`
  * `.unwatch()`
  * `.message()`

In place of the removed methods, you should use the new `.publish()` method, or the low-level [sails.sockets](http://sailsjs.com/documentation/reference/web-sockets/sails-sockets) methods.  Keep in mind that unlike `.message()`, `.publish()` does _not_ wrap your data in an envelope containing the record ID, so you'll need to include that as part of the data if it's important.

## Replacing custom blueprints

While it is no longer possible to add a file to `api/blueprints` that will automatically be used as a blueprint action for all models, this behavior can be replicated in several ways.

One way is to add a route like `'POST /:model': 'SharedController.create'` to the bottom of your `config/routes.js` file, and then add the custom `create` blueprint to a `api/controllers/SharedController.js` file (or a `api/controllers/shared/create.js` standalone action).

Another option would be to add a `api/helpers/create.js` helper which takes a model name and dictionary of attributes as inputs (see the Helpers docs at http://sailsjs.com/docs/concepts/helpers), and call that helper from the related action for each model (e.g. `UserController.create`).


## Express 4

Sails 1.0 comes with an update to the internal Express server from version 3 to version 4 (thanks to some great work by [@josebaseba](http://github.com/josebaseba)).  This change is mainly about maintainability for the Sails framework, and should be transparent to your app.  However, there are a couple of differences worth noting:

* The `404`, `500` and `startRequestTimer` middleware are now built-in to every Sails app, and have been removed from the `sails.config.http.middleware.order` array.  If your app has an overridden `404` or `500` handler, you should instead override `api/responses/notFound.js` and `api/responses/serverError.js` respectively.
* Session middleware that was designed specifically for Express 3 (e.g. very old versions of `connect-redis` or `connect-mongo`) will no longer work, so you&rsquo;ll need to upgrade to more recent versions.

## Responses
 * `.jsonx()` is deprecated -- if you haven't customized a response, just delete it.  Otherwise, replace `res.jsonx()` with `res.json()`.
 * `res.negotiate()` is deprecated -- use a [custom response](http://sailsjs.com/documentation/concepts/custom-responses) instead.


## i18n

Sails 1.0 switches from using the [i18n](http://npmjs.org/package/i18n) to the lighter-weight [i18n-2](http://npmjs.org/package/i18n-2) module.  The overwhelming majority of users should see no difference in their apps.  However, if you&rsquo;re using the `sails.config.i18n.updateFiles` option, be aware that this is no longer supported -- instead, locale files will _always_ be updated in development mode, and _never_ in production mode.  If this is a problem or you&rsquo;re missing some other feature from the i18n module, you can install [sails-hook-i18n](http://npmjs.org/package/sails-hook-i18n) to revert to pre-Sails-1.0 functionality.

## WebSockets

The `sails-hook-sockets` hook that is installed by default with new Sails apps now uses a newer version of Socket.io.  See the [Socket.io changelog](https://github.com/socketio/socket.io/blob/master/History.md#150--2016-10-06) for a full update, but one thing to keep in mind is that socket IDs no longer have `/#` prepended to them by default.