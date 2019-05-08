# AuthenticationService

[![npm version](https://img.shields.io/npm/v/@feathersjs/authentication.png?style=flat-square)](https://www.npmjs.com/package/@feathersjs/authentication)
[![Changelog](https://img.shields.io/badge/changelog-.md-blue.png?style=flat-square)](https://github.com/feathersjs/feathers/blob/master/packages/authentication/CHANGELOG.md)

```
$ npm install @feathersjs/authentication --save
```

The `AuthenticationService` is a [Feathers service](../services.md) that allows to register different authentication strategies and manage [JSON web tokens (JWT)](https://jwt.io/).

```js
const { AuthenticationService, JwtStrategy } = require('@feathersjs/authentication');

module.exports = app => {
  const authService = new AuthenticationService(app);

  service.register('jwt', new JwtStrategy());

  app.use('/authentication', authService);
}
```

## Configuration

## app.get('defaultAuthentication')

After registering an authentication service, it will set the `defaultAuthentication` property on the application to its configuration name (see `configKey` below) if it does not exist. `app.get('defaultAuthentication')` will be used by other parts of Feathers authentication to access the authentication service if it is not otherwise specified.

## constructor(app)

`new AuthenticationService(app, configKey = 'authentication')` initializes a new authentication service with the [Feathers application]() instance and a `configKey` which is the name of the configuration property to use via [app.get()](../application.md#get-name) (default: `app.get('authentication')`). Upon initialization it will also update the configuration with the [default settings](#configuration).

## configuration

`service.configuration` returns a copy of current value of `app.get(configKey)` (default: `app.get('authentication')`). It is usually initialized through an `authentication` property in a [configuration file](../configuration.md) (default: `config/default.json`) which must at least contain a `secret` property. The default configuration is:

```js
{
  entity: 'user', // The name of the entity. Can be `null` when not used
  service: 'users', // The service to retrieve the entity from
  jwtOptions: {
    header: { typ: 'access' }, // by default is an access token but can be any type
    audience: 'https://yourdomain.com', // The resource server where the token is processed
    issuer: 'feathers', // The issuing server, application or resource
    algorithm: 'HS256',
    expiresIn: '1d'
  },
  jwt: {
    header: 'Authorization', // The HTTP header value
    schemes: [ 'Bearer', 'JWT' ] // The header schemes to use (e.g. `Bearer <token>`)
  }
}
```

`jwt` contains the configuration for the [JWT authentication strategy](./jwt.md). `jwtOptions` can take all options available for the [node-jsonwebtoken package](https://github.com/auth0/node-jsonwebtoken).

> *Note:* `typ` in the `header` options is not a typo, it is part of the [JWT JOSE header specification](https://tools.ietf.org/html/rfc7519#section-5).

## register(name, strategy)

`service.register(name, strategy)` registers an [authentication strategy](./strategy.md) under `name` and calls the strategy methods that are implement.

## getStrategies(...names)

`service.getStrategies(...names)` returns the [authentication strategies](#authentication-strategies) that exist for a list of names. The returned array may include `undefined` values if the strategy does not exist.

## createAccessToken(payload)

`authService.createAccessToken(payload, [options, secret])` creates a new JWT access token with `payload` using [configuration.jwtOptions](#configuration) merged with `options` (optional). It will either use `configuration.secret` or the optional `secret` to sign the JWT. Returns a promise that resolves with the accessToken or throw an error.

> __Note:__ Normally, it is not necessary to call this method directly. Calling [authService.create(data, params)](#create-data-params) will take care of creating the correct access token.

```js
const token = await app.service('authentication').createAccessToken({
  permission: 'admin'
});
```

## verifyAccessToken(accessToken)

`authService.verifyAccessToken(accessToken, [options, secret])` verifies the JWT `accessToken` using `configuration.jwtOptions` merged with `options` (optional). Will either use `configuration.secret` or the optional `secret` to verify the JWT. Returns a promise that resolves with the encoded payload or throw an error.

## getTokenOptions(authResult, params)

Return the options for creating a new access token based on the return value from calling [authService.authenticate()](#authenticate-data-params-strategies). Called internally on [authService.create()](#create-data-params). Will try to set the JWT `subject` to the entity (user) id if it is available. This will be used by the [JWT strategy](./jwt.md) to populate `params[entity]` (usually `params.user`). Returns a promise that resolves with the token options.

## getPayload(authResult, params)

Returns the payload for an authentication result and parameters. Called internally on [.create](#create-data-params). Returns a promise that either contains `params.payload` or an empty object (`{}`).

## authenticate(data, params, ...strategies)

Run [.authenticate()](#authenticate-data-params) for the given `strategies` with `data` and `params`. Will return the value of the first strategy that didn't throw an error or the first error that ocurred. If `data.strategy` is set to a strategy name and it is not included in `strategies`, an error will be thrown.

## parse(req, res, ...strategies)

Parse a [NodeJS HTTP request](https://nodejs.org/api/http.html#http_class_http_incomingmessage) and [response](https://nodejs.org/api/http.html#http_class_http_serverresponse) for authentication information using `strategies` calling [each strategies `.parse()` method](#parse-req-res). Will return the value of the first strategy that didn't return `null`.

## setup(path, app)

Verifies the [configuration](#configuration) and makes sure that

- A `secret` has been set
- If `entity` is not `null`, check if the entity service is available and make sure that either the `entityId` configuration or the `entityService.id` property is set.
- Register internal hooks to send events and keep real-time connections up to date. All custom hooks should be registered at this time.

## create(data, params)

## remove(id, params)

Should be called with `id` set to `null` or to the authenticated JWT. Will verify `params.authentication` and emit the [`logout` event]() if successful.

## Customization

The `AuthenticationService` can be customized like any other ES6 class:

```js
const { AuthenticationService } = require('@feathersjs/authentication');

class MyAuthService extends AuthenticationService {
  async getPayload(authResult, params) {
    // Call original `getPayload` first
    const payload = await super.getPayload(authResult, params);
    const { user } = authResult;

    if (user && user.permissions) {
      payload.permissions = user.permissions;
    }

    return payload;
  }
}

app.use('/authentication', new MyAuthService(app));
```

Things to be aware of when extending the authentication service:

- When implementing your own `constructor`, always call `super(app, configKey)`
- When overriding a method, calling `super.method` and working with its return value is recommended unless you are certain your custom method behaves exactly the same way, otherwise things may no longer work as expected.
- When extending `setup`, `super.setup(path, app)` should always be called, otherwise events and real-time connection authentication will no longer work.

## Events

### app.on('login')

### app.on('logout')