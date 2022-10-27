# Back-Channel Logout (Beta)

> Note: This is a Beta branch to test Back-Channel Logout and should not be used in production.

## Install

```bash
npm i "auth0/express-openid-connect#back-channel-logout"
```

## Basic Setup

Load the following environment variables:

```bash
ISSUER_BASE_URL=https://YOUR_DOMAIN
CLIENT_ID=YOUR_CLIENT_ID
BASE_URL=https://YOUR_APPLICATION_ROOT_URL
SECRET={A_LONG_RANDOM_VALUE}
```

Configure the SDK with `backchannelLogout` enabled. You will also need a session store (like Redis) - you can use any `express-session` compatible store.

```js
// index.js
const { auth } = require('express-openid-connect');
const { createClient } = require('redis');
const RedisStore = require('connect-redis')(auth);

// redis@v4
let redisClient = createClient({ legacyMode: true });
redisClient.connect().catch(console.error);

app.use(
  auth({
    idpLogout: true,
    backchannelLogout: {
      store: new RedisStore({ client: redisClient }),
    },
  })
);
```

If you're already using a session store for stateful sessions you can just reuse that.

```js
app.use(
  auth({
    idpLogout: true,
    session: {
      store: new RedisStore({ client: redisClient }),
    },
    backchannelLogout: true,
  })
);
```

### This will:

- Create the handler `/backchannel-logout` that you can register with your ISP.
- On receipt of a valid Logout Token, the SDK will store an entry by `sid` (Session ID) and an entry by `sub` (User ID) in the `backchannelLogout.store` - the expiry of the entry will be set to the duration of the session (this is customisable using the [onLogoutToken](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/index.d.ts#L522) config hook)
- On all authenticated requests, the SDK will check the store for an entry that corresponds with the session's ID token's `sid` or `sub`. If it finds a corresponding entry it will invalidate the session and clear the session cookie. (This is customisable using the [isLoggedOut](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/index.d.ts#L536) config hook)
- If the user logs in again, the SDK will remove the `sub` entry in the Back-Channel Logout store to ensure they are not logged out immediately (this is customisable using the [onLogin](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/index.d.ts#L550) config hook)

## Resources

- The config options are [documented here](https://github.com/auth0/express-openid-connect/blob/backchannel-logout/index.d.ts#L500)

## Customizations

The SDK has a default implementation of invalidating the session upon receipt of a logout token. But it is also flexible enough to support other ways of doing this with different tradeoffs.

### Default: Storing a logout entry for `sid` (if present) and one for `sub` (if present).

See above for how this works.

Running example: [backchannel-logout](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/examples/backchannel-logout.js)

- run it using `npm run start:example -- backchannel-logout`
- login to the mock Identity Provider using any credentials
- issue a Back-Channel Logout by visiting `/logout-token` and clicking the button

### Match the application's session identifier to the IDPs `sid`

If you are using a custom session store, you can change your session identifier to match your identity provider's `sid` then simply delete the session on receipt of a logout token which contains your IDP's `sid`.

Running example: [backchannel-logout-custom-genid](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/examples/backchannel-logout-custom-genid.js)

### Query the session store for matching sessions

If you are using a custom session store upon which you can make complex queries, like searching entries for `sid` or `sub` claims. You can remove the relevant sessions from the store upon receipt of a logout token that contains a `sid` or a `sub`

Running example: [backchannel-logout-custom-genid](https://github.com/auth0/express-openid-connect/blob/back-channel-logout/examples/backchannel-logout-custom-query-store.js)
