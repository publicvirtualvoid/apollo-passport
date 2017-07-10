# apollo-passport

Full stack Apollo and PassportJS integration, inspired by Meteor Accounts.

[![npm](https://img.shields.io/npm/v/apollo-passport.svg?maxAge=2592000)](https://www.npmjs.com/package/apollo-passport) [![Circle CI](https://circleci.com/gh/apollo-passport/apollo-passport.svg?style=shield)](https://circleci.com/gh/apollo-passport/apollo-passport) [![Coverage Status](https://coveralls.io/repos/github/apollo-passport/apollo-passport/badge.svg?branch=master)](https://coveralls.io/github/apollo-passport/apollo-passport?branch=master) ![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)

Copyright (c) 2017 by Gilad Shoham & Gadi Cohen, released under the MIT license.

## IMPORTANT NOTICE
This package is named apollo-passportjs (instead of apollo-passport) in the npm.
I did this in order to be able to release it to npm.
The original author is not responding so i can't use the original name.

## IMPORTANT NOTICE (DEPRECATION)
This fork is maintained by Gilad Shoham.

I will make small fixes to this fork but main work will be done under
[`js-accounts`](https://github.com/js-accounts/accounts) framework.

You can still use `apollo-passport` if you need something in the meantime,
but all future work continues in `js-accounts`.

## Features

* Super fast start with optional, opinionated resolvers for common tasks and databases (see below).
* JSON Web Tokens (JWTs) for stateless "sessions" making database user lookups on every query optional.

* User interaction via GraphQL, not the framework.

  * Great for SPAs (single page apps) - no reloading or redirects to login; visible progress hints in UI.
  * Re-uses your existing transports.
  * No need for cookies and a cookie-free domain.


## New Features in this fork (Highlights)
  * Add option to define input apUserInput (outside) for creating new users with your desired fields
  * Add account verification token during create user
  * Add apVerifyAccount mutation to verify the account
  * Add recoverPasswordRequest mutation to create reset password token
  * Add options to pass hooks method (onCreateUserEnd, onBeforeStoreRegisteredUser, onRecoverPasswordRequestEnd, onVerifyAccountEnd, onRecoverPasswordEnd, onLoginEnd) (for example to send verification emails)
  * Improve errors format (Add error code)
  * Allow users without services to register even if their email already exist (Merge with existing user) for case that the user added from outside and not really registered
  # Align user schema (email field) with passport recommended structure from [here](http://passportjs.org/docs/profile)
  # Add register date during merge with existing user

## In Development

I'm still writing this.  Not everything mentioned in the README exists yet.  Not everything may work.  Most importantly, until a 1.0.0 release, NO SECURITY AUDIT HAS TAKEN PLACE.

Also, I probably won't have time to support this ;)  I'm using this, and it will work for whatever I need it for, but I'm hoping that anyone who uses this - especially at this stage - is interested in actively contributing to the project.  I hope to create a good starting point for community development.

Lastly, this is my first time using passport, apollo/graphql and JWT, so PRs for better practices are welcome.

**Bugs, feature requests, etc**

Top priority will be given to high quality PRs.  You can still help by reporting bugs, requesting features, etc, as long as you have realistic expectations :)  Consideration will be given to the number of users affected: See [open issues sorted by thumbs-up](https://github.com/GiladShoham/apollo-passport/issues?q=is%3Aissue+is%3Aopen+sort%3Areactions-%2B1-desc).

## Getting Started

Inspired by Meteor's account system, apollo-passport (optionally) comes with everything you need to get started quickly: an opinionated database structure, resolvers for various databases, and the pre-built UI components (just for react, for now) to interact with the user and even configure provider settings.

The example below shows the most common options, and may be customized with:

* Appropriate [database driver](https://www.npmjs.com/browse/keyword/apollo-passport-database-driver), e.g. `apollo-passport-rethinkdbdash`.
* Appropriate [UI framework](https://www.npmjs.com/browse/keyword/apollo-passport-ui-framework), e.g. `apollo-passport-react`.
* [Augmented strategies](https://www.npmjs.com/browse/keyword/apollo-passport-strategy), e.g. `apollo-passport-local`.
* Regular [passport strategies](http://passportjs.org/) by class, e.g. `oath2:facebook`.

```sh
# Typical packages.  Choose database, strategies, ui from the list above.
$ npm i --save apollo-passportjs \
  apollo-passport-local-strategy \
  apollo-passport-mongodb-driver \
  apollo-passport-react

# Other passport strategies (that don't have "augmented" apollo versions)
$ npm i --save passport-facebook
```

**Server entry point**

Note: the server side requires a `ROOT_URL` to be set.  This can be done via 1) environment variable, 2) a global / define, or 3) by passing a ROOT_URL option to `new ApolloPassport(options)`.

```js
import ApolloPassport from 'apollo-passportjs';
import MongoDriver from 'apollo-passport-mongodb-driver';
import { Strategy as LocalStrategy } from 'passport-local';
import { Strategy as FacebookStrategy } from 'passport-facebook';

// However you usually create your mongodb instance
const m = await MongoClient.connect(`mongodb://${host}:${port}/${name}`);

const apolloPassport = new ApolloPassport({
  db: new MongoDriver(m),          // "m" is your mongodb instance
  jwtSecret: 'my special secret',   // will be optional/automatic in the future
  authPath: '/ap-auth'             // default: '/ap-auth', changing untested
});

// Pass the class, not the instance (i.e. no NEW), and no options for defaults
// Make sure you setup strategies BEFORE calling getSchema, getResolvers below.
apolloPassport.use('local', LocalStrategy /*, options */);

// Example oauth2 strategy.  Meteor Accounts sytle configuration via UI coming soon...
// You need to authorize on facebook with http://.../ap-auth/facebook/callback
apolloPassport.use('oauth2:facebook', FacebookStrategy, {
  clientID: '403859966407266',
  clientSecret: 'fd3ec904596e0b775927a1052a3f7165',
  // What permissions to request for this user
  // https://developers.facebook.com/docs/facebook-login/permissions/overview
  scope: [ 'public_profile', 'email' ],
  // Which fields to request automatically on login
  // https://developers.facebook.com/docs/graph-api/reference/v2.5/user
  profileFields: [
    'id', 'email',
    'first_name', 'middle_name', 'last_name',
    'gender', 'locale'
  ]
});

// Merge these into your Apollo config however you usually do...
const apolloOptions = {
  schema: apolloPassport.schema(),
  resolvers: apolloPassport.resolvers()
};

// Augment apolloServer's entry point
app.use('/graphql', apolloServer(apolloPassport.wrapOptions(apolloOptions)));

// And add an entry-point for apollo passport.
app.use('/ap-auth', apolloPassport.expressMiddleware());
```

**Client config**

```js
// Configure Apollo
import ApolloClient, { createNetworkInterface } from 'apollo-client';
import ApolloPassport from 'apollo-passportjs/lib/client';
import ApolloPassportLocal from 'apollo-passport-local/lib/client';
import apMiddleware from 'apollo-passportjs/lib/client/middleware';

const networkInterface = createNetworkInterface('/graphql');
networkInterface.use([ apMiddleware ]);

const apolloClient = new ApolloClient({ networkInterface });
const apolloPassport = new ApolloPassport({ apolloClient });

apolloPassport.use('local', ApolloPassportLocal);

// Optional, if you use Redux... (combine with apollo's reducers & middleware)
const store = createStore(
  combineReducers({
    apollo: apolloClient.reducer(),
    auth: apolloPassport.reducer()
  }),
  applyMiddleware(
    apolloClient.middleware(),
    apolloPassport.middleware()
  )
);

export { apolloClient, apolloPassport };
```

**Client usage**:

```sh
$ npm i --save apollo-passport-react
```

```js
import { LoginButtons } from 'apollo-passport-react';
import 'apollo-passport-react/style/meteor.less';

// From the file above...
import { apolloPassport } from '../../../lib/apollo';

const SomewhereInMyApp = () => (
  <LoginButtons apolloPassport={apolloPassport} />
);
```

See [apollo-passport-react](https://www.npmjs.com/package/apollo-passport-react) for more details.

## Things to know

* During client load, a GraphQL query is sent to the server.  Consider enabling **query batching**.
* As mentioned above, a `ROOT_URL` is required.  It's used to auto-generate the callbackUrl if none is specific.  By default, http://www.blah.com/ap-auth/facebook/callback (Meteor-style guided setup coming soon).

[Let us know](https://github.com/GiladShoham/apollo-passport/issues/new) of any gotchas you come across so we can document them.

## API

### Server

#### new ApolloPassport(options)

Instantiates a new ApolloPassport instance for your app, with the given options.

Note: a number of options mention that are optional if custom `verify` functions are given.  This is gradually being phased out as we'd prefer to handle this on a framework level for consistency.

**Required Options**

* **db**: <ApolloPassportDBDriver>

  This is required for the default simple setup, but is not required if you provide your own passport verify functions.

* **jwtSecret**

  Required for now.  Will be created automatically and stored in the database in the future if not specified.  Also not required if the user providers their own verify functions.  Or we'll default to old style login tokens stored in the DB and fetch the user on each query.

**Customization Options**

* **mapUserToJWTProps**: default `user => ({ userId: user.id })`

  Specify your own function to store custom data in the JWT token, e.g. `isAdmin`, etc.  This will be available both on server resolvers under `context.auth` and on the client as `apolloPassport.getState().data` (directly or via subscription, redux or react HOC).

* **createTokenFromUser**: can be replaced if you know what you're doing (see src/index.js).

* **winston**

  We use [Winston](https://github.com/winstonjs/winston) for logging.  If you do too, you can pass in an existing `winston` instance and benefit from your existing transports.

#### apolloPassport.use('strategyName', StrategyClass, <options>, <verifyCallback>)

Self evident.  Use as per the examples above.

## Roadmap

* log user auths with ability for admins to see last x logins, failures, etc.
* let user see list of all login tokens per device and revoke access.
* service logins with same email are already automatically merged - need way to manually link/unlink accounts.
* pluggable interface to allow other packages to provide/replace e.g. react ui login.

## Why not use Meteor Accounts as a base?

* PassportJS is to the go-to-auth for node apps and has
  * 300+ authentication strategies
  * 8k stars on github
  * *Only* the auth layer, so easy to build upon
  * Strategies do a good job of normalizing data structure from different providers

* Meteor accounts
  * until Meteor 1.5, lots of deps on pure Meteor packages
  * need to restructure deps on Blaze, Mongo, DDP, pub/sub, etc.
  * less active development

**However**

Besides for PassportJS and JWTs, Meteor Accounts has a looot of stuff we
should consider re-using... UI, validation flow, etc.
