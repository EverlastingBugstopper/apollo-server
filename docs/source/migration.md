---
title: Migrating to Apollo Server 3
---

**Apollo Server 3 is generally available.** The focus of this major-version release is to provide a lighter, nimbler core library as a foundation for future features and improved extensibility.

**Many Apollo Server 2 users don't need to make any code changes to upgrade to Apollo Server 3**, especially if you use the "batteries-included" `apollo-server` library (as opposed to a [middleware-specific library](./integrations/middleware/)).

This document explains which features _do_ require code changes and how to make them.

> For a list of all breaking changes, [see the changelog](https://github.com/apollographql/apollo-server/blob/main/CHANGELOG.md).



## Bumped dependencies

### Node.js

**Apollo Server 3 supports Node.js 12 and later.** (Apollo Server 2 supports back to Node.js 6.) This includes [all LTS and Current versions at the time of release](https://nodejs.org/about/releases/).

If you're using an older version of Node, you should upgrade your Node runtime before upgrading to Apollo Server 3.

### `graphql`

Apollo Server has a peer dependency on [`graphql`](https://www.npmjs.com/package/graphql) (the core JS GraphQL implementation), which means you are responsible for choosing the version installed in your app.

**Apollo Server 3 supports `graphql` v15.3.0 and later.** (Apollo Server 2 supported `graphql` v0.12 through v15.)

If you're using an older version of `graphql`, you should upgrade it to the latest version before upgrading to Apollo Server 3.

## Removed integrations

Apollo Server 2 provides built-in support for subscriptions and file uploads via the `subscriptions-transport-ws` and `graphql-upload` packages, respectively. It also serves GraphQL Playground from its base URL by default.

Apollo Server 3 removes these built-in integrations, in favor of enabling users to provide their own mechanisms for these features.

You can reenable all of these integrations as they exist in Apollo Server 2.

### Subscriptions

Apollo Server 2 provides limited, built-in support for WebSocket-based GraphQL subscriptions via the `subscriptions-transport-ws` package. This integration doesn't work with Apollo Server's plugin system or Apollo Studio usage reporting.

Apollo Server 3 no longer contains this built-in integration. However, you can still use `subscriptions-transport-ws` for subscriptions if you depend on this implementation. Note that as with Apollo Server 2, this integration won't work with the plugin system or Studio usage reporting.

We hope to add more fully-integrated subscription support to Apollo Server in a future version.

#### Reenabling subscriptions

> **The `subscriptions-transport-ws` library is not actively maintained.** We recommend instead implementing subscriptions with the newer [`graphql-ws`](https://www.npmjs.com/package/graphql-ws) package (as described in [the subscriptions docs](../data/subscriptions/)).
>
> The steps below assume you want to continue using `subscriptions-transport-ws`, which requires fewer changes to existing code.

You cannot integrate the batteries-included `apollo-server` package with `subscriptions-transport-ws`; if you were using `apollo-server` with subscriptions, first eject to `apollo-server-express` (TODO: link to ejection doc; see #5097).

> *Note:* Currently, these instructions are only for the Express integration. It is likely possible to integrate `subscriptions-transport-ws` with other integrations. PRs to this migration guide are certainly welcome!

 1. Install `subscriptions-transport-ws` and `@graphql-tools/schema`.

        npm install subscriptions-transport-ws @graphql-tools/schema

 2. Import `SubscriptionServer` from `subscriptions-transport-ws`.

    ```javascript
    import { SubscriptionServer } from 'subscriptions-transport-ws';
    ```

 3. Import `makeExecutableSchema` from `@graphql-tools/schema`.

    > Your server may already be using `makeExecutableSchema`, so adding this
    > import may not be necessary.  More on why and how in the next step!

    ```javascript
    import { makeExecutableSchema } from '@graphql-tools/schema';
    ```

 4. Import the `execute` and `subscribe` functions from `graphql`.

    The `graphql` package should already be installed, so simply import them for
    usage:

    ```javascript
    import { execute, subscribe } from 'graphql';
    ```

    We will pass these to the creation of the `SubscriptionServer` in Step 9.

 5. Have an instance of `GraphQLSchema` available.

    The `SubscriptionServer` (which we'll initiate in a later step) doesn't
    accept `typeDefs` and `resolvers` directly; it instead only accepts
    an executable `GraphQLSchema` as `schema`. So we need to make one of those.
    We can then pass the same object to `new ApolloServer` instead of `typeDefs`
    and `resolvers` so it's clear that the same schema is being used in both
    places.

    > Your server may already pass a `schema` to `new ApolloServer`.  If it
    > does, this step can be skipped.  You'll use the your existing schema in
    > Step 8 below.

    ```javascript
    const schema = makeExecutableSchema({ typeDefs, resolvers });
    ```

    > While not necessary, this `schema` can be passed into the `ApolloServer`
    > constructor options, rather than `typeDefs` and `resolvers`:
    >
    > ```javascript
    > const server = new ApolloServer({
    >   schema,
    > });
    > ```

 6. Import Node.js's `createServer` from the `http` module.

    ```javascript
    import { createServer } from 'http';
    ```


7. Get an `http.Server` instance with the Express app, prior to `listen`-ing.

     In order to setup both the HTTP and WebSocket servers prior to listening,
     we'll need to get the `http.Server`.  Do this by passing the Express `app`
     to the `createServer` we imported from Node.js' `http` module.

     ```javascript
     // This `app` is the returned value from `express()`.
     const httpServer = createServer(app);
     ```

8. Create the `SubscriptionsServer`.

    ```javascript
    SubscriptionServer.create({
       // This is the `schema` created in Step 6 above.
       schema,

       // These were imported from `graphql` in Step 5 above.
       execute,
       subscribe,
    }, {
       // This is the `httpServer` created in Step 9 above.
       server: httpServer,

       // This `server` is the instance returned from `new ApolloServer`.
       path: server.graphqlPath,
    });
    ```

9. Finally, adjust the existing `listen`.

    Previously, most applications will be doing `app.listen(...)`.

    **This should be changed to `httpServer.listen(...)`** (same arguments) to
    start listening on the HTTP and WebSocket transports simultaneously.

### File uploads

Apollo Server 2 provides built-in support for file uploads via an outdated version of the `graphql-upload` library. Using an updated version of `graphql-upload` required you to disable this built-in support due to backward incompatible changes.

This built-in support is removed in Apollo Server 3. To use `graphql-upload`, you can choose an appropriate version and integrate it yourself. Note that `graphql-upload` does not support federation or every Node.js framework supported by Apollo Server.

To use `graphql-upload` with Apollo Server 3, see the [documentation on enabling file uploads in Apollo Server](../data/file-uploads/). Note that if you were using uploads with the  "batteries-included" `apollo-server` package, you must first eject to `apollo-server-express` (TODO: link to eject doc; see #5097).

### GraphQL Playground

By default, Apollo Server 2 serves the (now-[retired](https://github.com/graphql/graphql-playground/issues/1143)) [GraphQL Playground](https://github.com/graphql/graphql-playground) IDE from its base URL (unless it's running in production). You can override this default behavior with the `playground` option (for example, `playground:true` enables GraphQL Playground even in production).

Apollo Server 3 removes this option in favor of a new [landing page API](../integrations/plugins-event-reference/#renderlandingpage), which enables you to serve a custom landing page (or multiple landing pages). The default landing page is a splash page (TODO #5373: link to splash page docs), which is served in every environment but has a different appearance depending on the value of `NODE_ENV`. When `NODE_ENV` is not `production`, this splash page links to the Apollo Sandbox IDE.

#### Reenabling GraphQL Playground

You can continue to use GraphQL Playground by installing its associated plugin. You customize its behavior by passing options to the plugin instead of via the `playground` constructor option.

If your code did not specify the `playground` constructor option and you'd like to keep the previous behavior instead of trying the new splash page, you can do that as follows:

```javascript
import { ApolloServerPluginLandingPageGraphQLPlayground,
         ApolloServerPluginLandingPageDisabled } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    process.env.NODE_ENV === 'production'
      ? ApolloServerPluginLandingPageDisabled()
      : ApolloServerPluginLandingPageGraphQLPlayground(),
  ],
});
```

If your code passed `new ApolloServer({playground: true})`, you can keep the previous behavior with:

```javascript
import { ApolloServerPluginLandingPageGraphQLPlayground } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    ApolloServerPluginLandingPageGraphQLPlayground(),
  ],
});
```

If your code passed `new ApolloServer({playground: false})` you can keep the previous behavior with:

```javascript
import { ApolloServerPluginLandingPageDisabled } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    ApolloServerPluginLandingPageDisabled(),
  ],
});
```

If your code passed an options object like `new ApolloServer({playground: playgroundOptions})`, you can keep the previous behavior with:

```javascript
import { ApolloServerPluginLandingPageGraphQLPlayground } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    ApolloServerPluginLandingPageGraphQLPlayground(playgroundOptions),
  ],
});
```

#### Additional changes to GraphQL Playground

##### Specifying an `endpoint`

In Apollo Server 2, the default value of GraphQL Playground's `endpoint` option is determined in different ways by different Node.js framework integrations. In many cases, it's necessary to manually specify `playground: {endpoint}`.

In Apollo Server 3, the default endpoint used by GraphQL Playground is the browser's current URL. In many cases, this means that you don't have to specify `endpoint` any more. If your Apollo Server 2 app specified `playground: {endpoint}` (and you wish to continue using GraphQL Playground), try removing `endpoint` from the options passed to `ApolloServerPluginLandingPageGraphQLPlayground` and see if it works for you.

##### Specifying `settings`

In Apollo Server 2, the behavior of the `settings` GraphQL Playground option is surprising. If you don't explicitly pass `{playground: {settings: {...}}}` then GraphQL Playground always uses settings that are built into its React application (some of which can be adjusted by the user in their browser). However, if you pass _any object_ as `playground: {settings: {...}}`, [several default value overrides](https://github.com/apollographql/apollo-server/blob/70a431212bd2d07d68c962cb5ded63ecc6a21963/packages/apollo-server-core/src/playground.ts#L38-L46) take effect.

This confusing behavior is removed in Apollo Server 3. All `settings` use default values from the GraphQL Playground React app if they aren't specified in the `settings` option to `ApolloServerPluginLandingPageGraphQLPlayground`.

If your app does pass in `playground: {settings: {...}}` and you want to make sure the settings used in your GraphQL Playground do not change, you should copy any relevant settings from [the Apollo Server 2 code](https://github.com/apollographql/apollo-server/blob/70a431212bd2d07d68c962cb5ded63ecc6a21963/packages/apollo-server-core/src/playground.ts#L38-L46) into your app.

For example, you could replace:

```javascript
new ApolloServer({playground: {settings: {'some.setting': true}}})
```

with:

```javascript
import { ApolloServerPluginLandingPageGraphQLPlayground } from 'apollo-server-core';
new ApolloServer({
  plugins: [
    ApolloServerPluginLandingPageGraphQLPlayground({
      'some.setting': true,
      'general.betaUpdates': false,
      'editor.theme': 'dark' as Theme,
      'editor.cursorShape': 'line' as CursorShape,
      'editor.reuseHeaders': true,
      'tracing.hideTracingResponse': true,
      'queryPlan.hideQueryPlanResponse': true,
      'editor.fontSize': 14,
      'editor.fontFamily': `'Source Code Pro', 'Consolas', 'Inconsolata', 'Droid Sans Mono', 'Monaco', monospace`,
      'request.credentials': 'omit',
    }),
  ],
});
```


## Removed constructor options

The following `ApolloServer` constructor options have been removed in favor of other features or configuration methods.

### `extensions`

Apollo Server 3 removes support for the `graphql-extensions` API, which was used to extend Apollo Server's functionality. This API has numerous limitations relative to the [plugins API](./integrations/plugins/) introduced in Apollo Server v2.2.0.

Unlike `graphql-extensions`, the plugins API enables cross-request state, and its hooks virtually all interact with the same `GraphQLRequestContext` object.

If you've written your own extensions (passed to `new ApolloServer({extensions: ...})`), you should [rewrite them as plugins](./integrations/plugins/) before upgrading to Apollo Server 3.


### `engine`

"Engine" is a previous name of [Apollo Studio](https://www.apollographql.com/docs/studio/). Prior to Apollo Server v2.18.0, you passed the `engine` constructor option to configure how Apollo Server communicates with Studio.

In later versions of Apollo Server (including Apollo Server 3), you instead provide this configuration via a combination of the `apollo` constructor option, plugins, and `APOLLO_`-prefixed environment variables.

If your project still uses the `engine` option, see [Migrating from the `engine` option](../v2/migration-engine-plugins) before upgrading to Apollo Server 3.

### `schemaDirectives`

In Apollo Server 2, you can pass `schemaDirectives` to `new ApolloServer` alongside `typeDefs` and `resolvers`. These arguments are all passed through to the [`makeExecutableSchema` function](https://www.graphql-tools.com/docs/generate-schema/#makeexecutableschemaoptions)  from the `graphql-tools` package. `graphql-tools` now considers `schemaDirectives` to be a [legacy feature](https://www.graphql-tools.com/docs/legacy-schema-directives/).

In Apollo Server 3, the `ApolloServer` constructor now only passes `typeDefs`, `resolvers`, and `parseOptions` through to `makeExecutableSchema`.

To provide other arguments to `makeExecutableSchema` (such as `schemaDirectives` or its replacement `schemaTransforms`), you can call `makeExecutableSchema` yourself and pass its returned schema as the `schema` constructor option.

That is, you can replace:

```javascript
new ApolloServer({
  typeDefs,
  resolvers,
  schemaDirectives,
});
```

with:

```javascript
import { makeExecutableSchema } from '@graphql-tools/schema';

new ApolloServer({
  schema: makeExecutableSchema({
    typeDefs,
    resolvers,
    schemaDirectives,
  }),
});
```

> In Apollo Server 2, there are subtle differences between providing a schema with `schema` versus providing it with `typeDefs` and `resolvers`. For example, the automatic definition of the `@cacheControl` directive is added only in the latter case. These differences are removed in Apollo Server 3 (for example, the definition of the `@cacheControl` directive is _never_ automatically added).

### `tracing`

In Apollo Server 2, the `tracing` constructor option enables a trace mechanism implemented in the `apollo-tracing` package. This package uses a comparatively inefficient JSON format for execution traces returned via the `tracing` GraphQL response extension. The format is consumed only by the deprecated `engineproxy` and GraphQL Playground. It is _not_ the tracing format used for Apollo Studio usage reporting or federated inline traces.

The `tracing` constructor option is removed in Apollo Server 3. The `apollo-tracing` package has been deprecated and is no longer being published.

If you rely on this deprecated trace format, you might be able to use the old version of `apollo-server-tracing` directly:

```javascript
new ApolloServer({
  plugins: [
    require('apollo-tracing').plugin()
  ]
});
```

> This workaround has not been tested! If you need this to work and it doesn't, please file an issue and we will investigate a fix to enable support in Apollo Server 3.

### `cacheControl`

In Apollo Server 2, [cache policy support](../performance/caching/) is configured via the `cacheControl` constructor option. There are several improvements to the semantics of cache policies in Apollo Server 3, as well as changes to how caching is configured.

The `cacheControl` constructor option is removed in Apollo Server 3. To customize cache control, you instead manually install the cache control plugin (TODO #5374: link) and provide custom options to it.

For example, if you currently provide `defaultMaxAge` and/or `calculateHttpHeaders` to `cacheControl` like so:

```javascript
new ApolloServer({
  cacheControl: {
    defaultMaxAge,
    calculateHttpHeaders,
  },
});
```

You now provide them like so:

```javascript
import { ApolloServerPluginCacheControl } from 'apollo-server-core';

new ApolloServer({
  plugins: [
    ApolloServerPluginCacheControl({
      defaultMaxAge,
      calculateHttpHeaders,
    }),
  ],
})
```

If you currently pass `cacheControl: false` like so:

```javascript
new ApolloServer({
  cacheControl: false,
});
```

You now install the disabling plugin like so:

```javascript
import { ApolloServerPluginCacheControlDisabled } from 'apollo-server-core';

new ApolloServer({
  plugins: [
    ApolloServerPluginCacheControlDisabled(),
  ],
})
```

In Apollo Server 2, `cacheControl: true` was a shorthand for setting `cacheControl: {stripFormattedExtensions: false, calculateHttpHeaders: false}`. If you either passed `cacheControl: true` or explicitly passed `stripFormattedExtensions: false`, Apollo Server 2 would include a `cacheControl` response extension inside your GraphQL response. This was used by the deprecated `engineproxy` server. Support for writing this response extension has been removed from Apollo Server 2. This allows for a more memory-efficient cache control plugin implementation.

In Apollo Server 2, definitions of the `@cacheControl` directive (and the `CacheControlScope` enum that it uses) were sometimes automatically inserted into your schema. (Specifically, they were added if you defined your schema with the `typeDefs` and `resolvers` options, but not if you used the `modules` or `schema` options or if you were a federated gateway. Passing `cacheControl: false` did not stop the definitions from being inserted!) In Apollo Server 3, these definitions are never automatically inserted.

So **if you use the `@cacheControl` directive in your schema, you should add these definitions to your schema**:

```graphql
enum CacheControlScope {
  PUBLIC
  PRIVATE
}

directive @cacheControl(
  maxAge: Int
  scope: CacheControlScope
  inheritMaxAge: Boolean
) on FIELD_DEFINITION | OBJECT | INTERFACE | UNION
```

(You may add them to your schema in Apollo Server 2 before upgrading if you'd like.)

In Apollo Server 2, plugins that want to change the operation's overall cache policy can overwrite the field `requestContext.overallCachePolicy`. In Apollo Server 3, that field is considered read-only, but it does have new methods to mutate its state. So you should replace:

```javascript
requestContext.overallCachePolicy = { maxAge: 100 };
```

with:

```javascript
requestContext.overallCachePolicy.replace({ maxAge: 100 });
```

(You may also want to consider using `restrict` instead of `replace`; this method only allows `maxAge` to be reduced and only allows `scope` to change from `PUBLIC` to `PRIVATE`.)

In Apollo Server 2, fields returning a union type are treated similarly to fields returning a scalar type: `@cacheControl` on the type itself is ignored, and `maxAge` if unspecified is inherited from its parent in the operation (unless it is a root field) instead of defaulting to `defaultMaxAge` (which itself defaults to 0). In Apollo Server 3, fields returning a union type are treated similarly to fields returning an interface or object type: `@cacheControl` on the type itself is honored, and `maxAge` if unspecified defaults to `defaultMaxAge`. If you were relying on the inheritance behavior, you can specify `@cacheControl(maxAge: ...)` explicitly on your union types or union-returning fields, or you can use the new `@cacheControl(inheritMaxAge: true)` feature on the union-returning field to restore the Apollo Server 2 behavior. If your schema contained `union SomeUnion @cacheControl(...)`, that directive will start having an effect when you upgrade to Apollo Server 3.

In Apollo Server 2, the `@cacheControl` is honored on type definitions but not on type extensions. That is, if you write `type SomeType @cacheControl(maxAge: 123)` it takes effect but if you write `extend type SomeType @cacheControl(maxAge: 123)` it does not take effect. In Apollo Server 3, `@cacheControl` is honored on object, interface, and union extensions. If your schema accidentally contained `@cacheControl` on an `extend`, that directive will start having an effect when you upgrade to Apollo Server 3.

### `playground`

See [GraphQL Playground](#graphql-playground).

## Removed exports

In Apollo Server 2, `apollo-server` and framework integration packages such as `apollo-server-express` import many symbols from third-party packages and re-export them. This effectively ties the API of Apollo Server to a specific version of those third-party packages and makes it challenging to upgrade to a newer version or for you to upgrade those packages yourself.

In Apollo Server 3, most of these "re-exports" are removed. If you want to use these exports, you should import them directly from their originating package

### Exports from `graphql-tools`

Apollo Server 2 exported every symbol exported by [`graphql-tools`](https://www.graphql-tools.com/) v4. If you're importing any of the following symbols from an Apollo Server package, you should instead run `npm install graphql-tools@4.x` and import the symbol from `graphql-tools` instead. Alternatively, read the [GraphQL Tools docs](https://www.graphql-tools.com/docs/introduction) and find out which `@graphql-tools/subpackage` the symbol is exported from in more modern versions of GraphQL Tools.

- `AddArgumentsAsVariables`
- `AddTypenameToAbstract`
- `CheckResultAndHandleErrors`
- `DirectiveResolverFn`
- `ExpandAbstractTypes`
- `ExtractField`
- `FilterRootFields`
- `FilterToSchema`
- `FilterTypes`
- `GraphQLParseOptions`
- `IAddResolveFunctionsToSchemaOptions`
- `IConnectorCls`
- `IConnectorFn`
- `IConnector`
- `IConnectors`
- `IDelegateToSchemaOptions`
- `IDirectiveResolvers`
- `IEnumResolver`
- `IExecutableSchemaDefinition`
- `IFieldIteratorFn`
- `IFieldResolver`
- `IGraphQLToolsResolveInfo`
- `ILogger`
- `IMockFn`
- `IMockOptions`
- `IMockServer`
- `IMockTypeFn`
- `IMocks`
- `IResolverObject`
- `IResolverOptions`
- `IResolverValidationOptions`
- `IResolversParameter`
- `IResolvers`
- `ITypeDefinitions`
- `ITypedef`
- `MergeInfo`
- `MergeTypeCandidate`
- `MockList`
- `NextResolverFn`
- `Operation`
- `RenameRootFields`
- `RenameTypes`
- `ReplaceFieldWithFragment`
- `Request`
- `ResolveType`
- `Result`
- `SchemaDirectiveVisitor`
- `SchemaError`
- `TransformRootFields`
- `Transform`
- `TypeWithResolvers`
- `UnitOrList`
- `VisitTypeResult`
- `VisitType`
- `WrapQuery`
- `addCatchUndefinedToSchema`
- `addErrorLoggingToSchema`
- `addMockFunctionsToSchema`
- `addResolveFunctionsToSchema`
- `addSchemaLevelResolveFunction`
- `assertResolveFunctionsPresent`
- `attachConnectorsToContext`
- `attachDirectiveResolvers`
- `buildSchemaFromTypeDefinitions`
- `chainResolvers`
- `checkForResolveTypeResolver`
- `concatenateTypeDefs`
- `decorateWithLogger`
- `defaultCreateRemoteResolver`
- `defaultMergedResolver`
- `delegateToSchema`
- `extendResolversFromInterfaces`
- `extractExtensionDefinitions`
- `forEachField`
- `introspectSchema`
- `makeExecutableSchema`
- `makeRemoteExecutableSchema`
- `mergeSchemas`
- `mockServer`
- `transformSchema`

### Exports from `graphql-subscriptions`

Apollo Server 2 exports every symbol exported by [`graphql-subscriptions`](https://www.npmjs.com/package/graphql-subscriptions). If you are importing any of the following symbols from an Apollo Server package, you should instead run `npm install graphql-subscriptions` and import the symbol from `graphql-subscriptions` instead.

- `FilterFn`
- `PubSub`
- `PubSubEngine`
- `PubSubOptions`
- `ResolverFn`
- `withFilter`

### Exports from `graphql-upload`

Apollo Server 2 exports the `GraphQLUpload` symbol from (our fork of) `graphql-upload`. Apollo Server 3 no longer has built-in `graphql-upload` integration. See [the documentation on how to enable file uploads in Apollo Server 3](../data/file-uploads/).

### Exports related to GraphQL Playground

Apollo Server 2 exported a `defaultPlaygroundOptions` object, along with `PlaygroundConfig` and `PlaygroundRenderPageOptions` types to support the `playground` top-level constructor argument.

In Apollo Server 3, GraphQL Playground is one of several landing pages implemented via plugins, and there are no default options for it. The `ApolloServerPluginLandingPageGraphQLPlaygroundOptions` type exported from `apollo-server-core` plays a similar role to `PlaygroundConfig` and `PlaygroundRenderPageOptions`. See [the section on `playground` above](#graphql-playground) for more details on configuring GraphQL Playground in Apollo Server 3.

## Removed features

Several small features have been removed from Apollo Server 3.
### `ApolloServer.schema` field

Apollo Server 2 had a deprecated field `ApolloServer.schema` (which never worked when the server was a gateway). Apollo Server 3 does not contain this field. If you want to access your server's schema, you have a few options:

- Just construct the schema yourself (eg, with `const schema = makeExecutableSchema({typeDefs, resolvers})` from `@graphql-tools/schema`), pass it in as `new ApolloServer({schema})`, and use the same value you've passed in elsewhere. (Apollo Server 3 won't modify the schema you passed in.)
- Write a plugin that uses the [`serverWillStart` event](../integrations/plugins-event-reference/#serverwillstart) to learn the schema. Note that if your server is a gateway, this will receive the first schema loaded on startup but will not receive any subsequent schemas if it updates later.
- If your server is a gateway, register a callback with `gateway.onSchemaChange`. Note that this API has some inconsistent behavior; we are considering [adding an Apollo Server plugin event that receives all schema updates](https://github.com/apollographql/apollo-server/pull/5187) to resolve this.

### `apollo-server-testing`

Apollo Server 2 contained a package `apollo-server-testing`. This package was a very thin wrapper around the `server.executeOperation` method. As of Apollo Server 2.25.0, we no longer document this package and instead [document using `executeOperation` directly](../testing/testing/).

In Apollo Server 3, we no longer publish this package. It's possible that the Apollo Server 2 version of this package might work with Apollo Server 3 servers. If you do use `apollo-server-testing`, we suggest that you migrate to using `executeOperation` directly (with at least 2.25.0 of Apollo Server) before upgrading to Apollo Server 3.

If you used `apollo-server-testing` like so:

```javascript
const { createTestClient } = require('apollo-server-testing');
const { query, mutate } = createTestClient(server);
await query({ query: QUERY });
await mutate({ mutation: MUTATION });
```

then you can use `executeOperation` like so:

```javascript
await server.executeOperation({ query: QUERY });
await server.executeOperation({ query: MUTATION });
```

(Note that before Apollo Server 2.25.0, the `apollo-server-testing` functions allowed you to pass your operation as a string or a `DocumentNode` as returned from `gql`, but `executeOperation` only supported strings. As of v2.25.0, `executeOperation` takes `string` or `DocumentNode`, which makes the change shown above simple in all cases.)

### `apollo-datasource-rest`: `baseURL` override change

When you create a [`RESTDataSource`](../data/data-sources/) subclass, you need to provide its `baseURL`. This can be done via `this.baseURL = ...` in the constructor or `resolveURL`, or via `baseURL = ...` at the class level.

In Apollo Server 2 you can also provide `baseURL` via a getter like `get baseURL() { ... }`. Apollo Server 3 is compiled with TypeScript 4, which [no longer supports overriding a property with an accessor](https://devblogs.microsoft.com/typescript/announcing-typescript-4-0/#properties-overriding-accessors-and-vice-versa-is-an-error) so this is no longer allowed.

If you used a getter in order to provide a dynamic URL, like this:

```javascript
class MyDataSource extends RESTDataSource {
  get baseURL() {
    return someDynamicallyCalculatedURL();
  }
}
```

you can instead override `resolveURL`:

```javascript
class MyDataSource extends RESTDataSource {
  async resolveURL(request: RequestOptions) {
    if (!this.baseURL) {
      this.baseURL = someDynamicallyCalculatedURL();
    }
    return super.resolveURL(request);
  }
}
```

## Changed features

### Plugin API

#### Almost all plugin events are now async

In Apollo Server 2, some [plugin events](../integrations/plugins-event-reference/) are synchronous (their return value is not a `Promise`), and some are "maybe-asynchronous" (they could return a `Promise` if they wanted, but didn't have to). This meant that you couldn't do asynchronous work in the former events, and the typings for the latter events were somewhat complex.

In Apollo Server 3, almost all plugin methods are always asynchronous: they always return a `Promise` type. This includes [end hooks](../integrations/plugins/#end-hooks) as well. (The one exception is `willResolveField` and its end hook, which are always synchronous.)

In practice, this means that all of your plugin events should use `async` functions or methods. If you are using TypeScript, you will need to do this in order for your code to compile.

(In practice, Apollo Server generally uses `await` or `Promise.all` on the values returned by plugin methods rather than an explicit `.then`; these constructs accept both `Promise`s and normal values. So if you are not using TypeScript you may be able to get away with plugin methods that are synchronous for now.)

#### `willSendResponse` is called more consistently

In Apollo Server 2, some errors related to [persisted queries](../performance/apq/) invoke the `requestDidStart` and `didEncounterError` plugin events without invoking the `willSendResponse` event afterwards. In Apollo Server 3, any request that makes it far enough to invoke `requestDidStart` will also invoke `willSendResponse`. See [this PR](https://github.com/apollographql/apollo-server/pull/5287) for details.


#### Gateway interface renamed and simplified

In Apollo Server 2, the TypeScript type used for the `gateway` constructor option is called `GraphQLService`. In Apollo Server 3, it is called `GatewayInterface`. (For now, an identical interface named `GraphQLService` continues to be exported.)

This interface now requires the `stop` method to exist, requires `executor` to be async, and requires you to always pass `apollo` to `load`. (All recent versions of `@apollo/gateway` satisfy these stronger requirements.)

### Bad request errors more consistently return 4xx

In Apollo Server 2, some poorly-formatted requests could receive HTTP responses with a 5xx status (indicating a server error) rather than 4xx (indicating a client error). For example, this occurred in some integrations for a missing POST body or JSON parse error. In Apollo Server 3, these errors are handled in a more consistent manner across integrations and consistently return HTTP responses with a 4xx status.

### Extensions (custom details) on `ApolloError`

In Apollo Server 2, [error extensions](http://localhost:8000/data/errors/#including-custom-error-details) could be passed either as the third argument to the constructor, *or* as the `extensions` option on the third argument. That is, `new ApolloError(message, code, {key: 'value'})` was equivalent to `new ApolloError(message, code, {extensions: {key: 'value'}})`. No other "options" were recognized on the third argument. In Apollo Server 3, only the first form is supported. (If you try to use the second form, the constructor will throw an error.) Replace any code using the second form with the first form; you can do this before upgrading.

In Apollo Server 2, an error extension `foo` would show up on an error object `e` both as `e.foo` and as `e.extensions.foo`; when serialized for JSON error responses, it would show up both as `extensions.foo` and `extensions.exception.foo`. In Apollo Server 3, it only shows up in one place: on `e.extensions.foo` on the error object, and in `extensions.foo` when serialized. See [the PR](https://github.com/apollographql/apollo-server/pull/5294) for more details.

### Mocking has changed due to `graphql-tools` upgrade

In Apollo Server 2, the `mocks` and `mockEntireSchema` constructor options are essentially implemented as follows:

```javascript
import { addMockFunctionsToSchema } from 'graphql-tools';  // v4.x

const { mocks, mockEntireSchema } = constructorOptions;
const schemaWithMocks = addMockFunctionsToSchema({
  schema,
  mocks:
    typeof mocks === 'boolean' || typeof mocks === 'undefined'
      ? {} : mocks,
  preserveResolvers:
    typeof mockEntireSchema === 'undefined' ? false : !mockEntireSchema,
});
```

In Apollo Server 3, we've upgraded from `graphql-tools` v4 to `@graphql-tools/mock` v8. The details of how mocking works in GraphQL Tools has changed, and the name of the function that implements it has changed too. Our implementation is as follows:

```javascript
import { addMocksToSchema } from '@graphql-tools/mock';  // v8.x

const { mocks, mockEntireSchema } = constructorOptions;
const schemaWithMocks = addMocksToSchema({
  schema,
  mocks: mocks === true || typeof mocks === 'undefined' ? {} : mocks,
  preserveResolvers:
    typeof mockEntireSchema === 'undefined' ? false : !mockEntireSchema,
});
```

So the name of the function has changed, and additionally, the semantics of the function has changed. You can [read the GraphQL Tools migration docs](https://www.graphql-tools.com/docs/mocking/#migration-from-v7-and-below) to see if you use any of the features of mocking that have changed between v4 and v8, and adjust the value of your `mocks` argument if so.

If you'd like to use features of `addMocksToSchema` that require passing more options to it than the three that `ApolloServer` passes through, just use it directly. To do this, run `npm install @graphql-tools/mock @graphql-tools/schema` in your app, and replace

```javascript
new ApolloServer({ typeDefs, resolvers, mocks });
```

with

```javascript
import { addMocksToSchema } from '@graphql-tools/mock';
import { makeExecutableSchema } from '@graphql-tools/schema';

new ApolloServer({
  schema: addMocksToSchema({
    schema: makeExecutableSchema({ typeDefs, resolvers }),
    mocks: // ...
    // ...
  }),
});
```


Alternatively, to keep the current behavior without changing your mock functions at all, you can continue to use `graphql-tools` v4. To do this, run `npm install graphql-tools@v4.x` in your app, and replace

```javascript
new ApolloServer({typeDefs, resolvers, mocks, preserveResolvers});
```

with

```javascript
import { makeExecutableSchema, addMockFunctionsToSchema } from 'graphql-tools';

new ApolloServer({
  schema: addMockFunctionsToSchema({
    schema: makeExecutableSchema({typeDefs, resolvers}),
    mocks,
    preserveResolvers:
      typeof mockEntireSchema === 'undefined' ? false : !mockEntireSchema,
  }),
});
```

### `apollo-server-caching` test suite helpers

In Apollo Server 2, the `apollo-server-caching` package exports functions like `testKeyValueCache`, `testKeyValueCache_Basic`, and `testKeyValueCache_Expiration` which define Jest test suites, and a `TestableKeyValueCache` type required for these test suites to be able to flush and close the cache. You could use this to define Jest test suites for your own implementation of the `KeyValueCache` interface, though this required some intricate Jest and TypeScript config to get working properly. (Or at least the README claims you could do this; the functions are not actually included in the package's `dist` directory but maybe you figured out how to pull them out of the `src/__tests__` directory? Anyway, you definitely could implement `TestableKeyValueCache` if you wanted.)

In Apollo Server 3, these functions and type are removed. Instead, a `runKeyValueCacheTests` function is exported which can be run in any test suite (without any Jest-specific behavior). See the `apollo-server-caching` README for more details.

## Changes to framework integrations

### `start()` now mandatory for non-serverless framework integrations

Apollo Server v2.22 introduced the `server.start()` method for non-serverless framework integrations (Express, Fastify, Hapi, Koa, Micro, and Cloudflare). Users of these integrations were encouraged to await a call to it immediately after creating an `ApolloServer` object:

```javascript
async function startServer() {
  const server = new ApolloServer({...});
  await server.start();
  server.applyMiddleware({ app });
}
```

If any plugin `serverWillStart` events throw, or if the server fails to load its schema properly (for example, if the server is a gateway and cannot load its schema), then `await server.start()` will throw. This allows you to ensure that Apollo Server has successfully loaded its configuration before you start listening for HTTP requests.

In Apollo Server 2, calling this method was optional. If you didn't call it, you could still call methods like `applyMiddleware` and start listening. If a startup failure occurred, all GraphQL requests would fail.

In Apollo Server 3, you *must* `await server.start()` immediately after `new ApolloServer` before calling `applyMiddleware` (or `getMiddleware` or `getHandler`, depending on the integration). (Note that you don't have to call it before calling the testing method `executeOperation`.) Otherwise, that method will throw.

This does not apply to the batteries-included `apollo-server` package (the server is started as part of the async method `server.listen()`) or to serverless framework integrations.

### Peer deps rather than direct deps

In Apollo Server 2, the `apollo-server-express`, `apollo-server-koa`, and `apollo-server-micro` packages has *direct* dependencies on `express`, `koa`, and `micro` respectively, and `apollo-server-fastify` and `apollo-server-hapi` has no dependency on `fastify` or `hapi` or `@hapi/hapi`.

In Apollo Server 3, these packages have *peer* dependencies on their corresponding framework packages. This means that you need to install a version of that package of your choice yourself in your app (though most likely that was already the case).
### CORS `*` is now the default

In Apollo Server 2, the default CORS configuration for most packages is to serve a `access-control-allow-origin: *` header for all responses.

However, this is not the case in Apollo Server 2 for `apollo-server-lambda`, `apollo-server-cloud-functions`, or `apollo-server-azure-functions`, which serve no CORS headers by default; or for `apollo-server-koa`, which serves an `access-control-allow-origin` header with a value matching the request's origin by default.

In Apollo Server 3, all implementations of `ApolloServer` that support CORS configuration have the same default of `access-control-allow-origin: *`.

If you use `apollo-server-lambda` or `apollo-server-cloud-functions` with the default `cors` configuration and you want to keep serving no CORS headers, call `server.createHandler({expressGetMiddlewareOptions: {cors: false}})` instead.

If you use `apollo-server-azure-functions` with the default `cors` configuration and you want to keep serving no CORS headers, call `server.createHandler({cors: false})` instead.

If you use `apollo-server-koa` with the default `cors` configuration and you want to keep serving a "reflected" `access-control-allow-origin`, call `server.getMiddleware({cors: {}})` instead.

Note that `apollo-server-micro` and `apollo-server-cloudflare` do not support CORS configuration in Apollo Server 2 or 3.

### `apollo-server-express`

In Apollo Server 2, `apollo-server-express` officially supports both the `express` framework and the older [`connect`](https://www.npmjs.com/package/connect) framework.

In Apollo Server 3, we no longer promise that we will continue to support `connect`. We do still run a test suite against it and we will try to not _accidentally_ break functionality under `connect`, but if future changes are easier to implement in an Express-only fashion, we reserve the right to break `connect` compatibility within Apollo Server 3.

### `apollo-server-lambda`

#### Now based on Express; `createHandler` arguments changed

In Apollo Server 2, `apollo-server-lambda` implements parsing AWS Lambda events and performing standard web framework logic inside the package itself. As Lambda added new capabilities, the logic had to adapt, and as Apollo Server added new features, they had to be implemented from scratch in `apollo-server-lambda`. Additionally, there was no way to add additional HTTP functionality (eg "middleware") to `apollo-server-lambda`.

In Apollo Server 3, `apollo-server-lambda` is implemented as a wrapper around `apollo-server-express`, using an [actively maintained package](https://www.npmjs.com/package/@vendia/serverless-express) to parse AWS Lambda events into Express requests. This simplifies the implementation and enables us to support new Lambda integrations such as Application Load Balancers. Additionally, because it now uses Express internally, you can add additional HTTP functionality via Express middleware using the new `expressAppFromMiddleware` option to `createHandler`.

As part of this, the arguments to `createHandler` have changed. Instead of taking only `cors` and `onHealthCheck`, it takes an `expressGetMiddlewareOptions` option, which is an object taking any of the options that the `apollo-server-express` `applyMiddleware` and `getMiddleware` can take (other than `app`). For example, instead of `server.createHandler({onHealthCheck})`, you now write `server.createHandler({expressGetMiddlewareOptions: {onHealthCheck}})`.

#### Handler is always async

In Apollo Server 2, the handler returned by `createHandler` can be called either as a function that takes a callback as a third argument, or (starting with Apollo Server v2.21.2) as an async function that returns a `Promise`.

In Apollo Server 3, the handler returned by `createHandler` is always an async function that returns a `Promise`; any callback passed in will be ignored.

All currently supported Lambda runtimes support async handlers; however, if you were wrapping the handler returned from `createHandler` in your own larger handler and passing a callback into it, this will no longer work. You should rewrite your outer handler to be an async function and `await` the `Promise` returned from the handler returned by `createHandler`.

For example, if your server looked like:

```javascript
const apolloHandler = server.createHandler();

exports.handler = function (event, context, callback) {
  doSomethingFirst();
  apolloHandler(event, context, (error, result) => {
    doSomethingLater();
    callback(error, result);
  });
}
```

you should change it to look like:

```javascript
const apolloHandler = server.createHandler();

exports.handler = async function (event, context) {
  doSomethingFirst();
  try {
    return await apolloHandler(event, context);
  } finally {
    doSomethingLater();
  }
}
```

### `apollo-server-cloud-functions`

In Apollo Server 2, `apollo-server-cloud-functions` implements standard web framework logic inside the package itself. Even though the Node API for Google Cloud Functions provides its request and response in the form of Express request and response objects, Apollo Server 2 had a bespoke implementation unrelated to `apollo-server-express` and did not support all features supported by `apollo-server-express`. Additionally, there was no way to add additional HTTP functionality (eg "middleware") to `apollo-server-cloud-functions`.

In Apollo Server 3, `apollo-server-cloud-functions` is implemented on top of `apollo-server-express` and supports all features supported by `apollo-server-express`. You can add additional HTTP functionality via Express middleware using the new `expressAppFromMiddleware` option to `createHandler`.

As part of this, the arguments to `createHandler` have changed. Instead of taking only `cors`, it takes an `expressGetMiddlewareOptions` option, which is an object taking any of the options that the `apollo-server-express` `applyMiddleware` and `getMiddleware` can take (other than `app`). For example, instead of `server.createHandler({cors})`, you now write `server.createHandler({expressGetMiddlewareOptions: {cors}})`.

### `apollo-server-fastify`

In Apollo Server 2, `apollo-server-fastify` supports Fastify v2 and did not support Fastify v3. There was no dependency or peer dependency making this requirement clear.

In Apollo Server 3, `apollo-server-fastify` supports Fastify v3. It has not been tested with versions of Fastify older than v3. There is a peer dependency on `fastify` v3.

### `apollo-server-hapi`

We are not certain precisely which versions of Hapi are supported by `apollo-server-hapi` Apollo Server 2, but we do know that only `@hapi/hapi` v20.1.2 and higher are supported by Node 16, and Apollo Server 2 was not tested with versions of `hapi` newer than v17.8.5. (Hapi's package name changed from `hapi` to `@hapi/hapi` between v18.1.0 and v18.2.0.)

In Apollo Server 3, `apollo-server-hapi` works with `@hapi/hapi` v20.1.2 and Node 16; it is not tested with older versions of Hapi. Note that the Hapi project believes that [all versions older than v20 have security issues](https://github.com/hapijs/hapi/issues/4114).