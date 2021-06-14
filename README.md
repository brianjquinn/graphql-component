![](https://github.com/ExpediaGroup/graphql-component/workflows/Build/badge.svg)

# GraphQL schema components.

This project is designed to faciliate componentized or modularized development of GraphQL schemas.

Read more about the idea [here](https://medium.com/expedia-group-tech/graphql-component-architecture-principles-homeaway-ede8a58d6fde).

`graphql-component` lets you build a schema progressively through a tree (faciliated through `imports`) of GraphQLComponent instances. Each GraphQLComponent instance encapsulates a GraphQL schema, specifically a `graphql-js` GraphQLSchema object.

Generally speaking, each instance of `GraphQLComponent` has reference to an instance of [`GraphQLSchema`](https://graphql.org/graphql-js/type/#graphqlschema). This instance of `GraphQLSchema` is built in a several ways, depending on the options passed to a given `GraphQLComponent`'s constructor.

* when a `GraphQLComponent` instance has `imports` (ie. other `GraphQLComponent` instances or component configuration objects) [graphql-tools stitchSchemas()](https://www.graphql-tools.com/docs/schema-stitching/) is used to create a "gateway" or aggregate schema that is the combination of the underlying imported schemas, and the typeDefs/resolvers passed to the root or importing `GraphQLComponent`
* when a `GraphQLComponent` has no imports, graphql-tools' `makeExecuteableSchema({typeDefs, resolvers})` is used to simply generate an executable GraphQL schema

An Apollo Federated schema can be constructed from 1 or more GraphQLComponent instances by passing the `federation: true` flag to the constructor.

### Running the examples

composition:
  * can be run with `npm run start-composition`

federation:
  * can be run with `npm run start-federation`

### Repository structure

- `lib` - the graphql-component code.
- `examples/composition` - a simple example of composition using `graphql-component`
- `examples/federation` - a simple example of building a federated schema using `graphql-component`

### Running examples:
* composition: `npm run start-composition`
* fedration: `npm run start-federation`
* go to `localhost:4000/graphql`
  * for composition this will bring up the GraphQL Playground for a plain old Apollo Server
  * for the federation example this will bring up the GraphQL Playground for an Apollo Federated Gateway

### Debug output

Generally enable debug logging with node environment variable `DEBUG=graphql-component:*`

# API

- `GraphQLComponent(options)` - the component class, which may also be extended. Its options include:
  - `types` - a string or array of strings representing typeDefs and rootTypes.
  - `resolvers` - an object containing resolver functions.
  - `imports` - an optional array of imported components for the schema to be merged with.
  - `context` - an optional object { namespace, factory } for contributing to context.
  - `directives` - an optional object containing custom schema directives.
  - `mocks` - a boolean (to enable default mocks) or an object to pass in custom mocks
  - `dataSources` - an array of data sources instances to make available on `context.dataSources` .
  - `dataSourceOverrides` - overrides for data sources in the component tree.
  - `federation` - enable building a federated schema (default: `false`).

- `GraphQLComponent.delegateToComponent(component, options)` - a wrapper function that utilizes `graphql-tools` `delegateToSchema()` to delegate the calling resolver's selection set to a root type field (`Query`, `Mutuation`) of another `GraphQLComponent`'s schema
  - `component` (instance of `GraphQLComponent`) - the component's whose schema will be the target of the delegated operation
  - `options` (`object`)
    - `operation` (optional, can be inferred from `info`): `query` or `mutation`
    - `fieldName` (optional, can be inferred if target field has same name as calling resolver's field): the target root type (`Query`, `Mutation`) field in the target `GraphQLComponent`'s schema
    - `context` (required) - the `context` object from resolver that calls `delegateToComponent`
    - `info` (required) - the `info` object from the resolver that calls `delegateToComponent`
    - `args` (`object`, optional) - an object literal whose keys/values are passed as args to the delegatee's target field resolver. By default, the resolver's args from which `delegateToComponent` is called will be passed if the target field has an argument of the same name. Otherwise, arguments passed via the `args` object will override the calling resolver's args of the same name.
    - `transforms` (optional `Array<Transform>`): Transform being a valid `graphql-tools` transform

  - please see `graphql-tools` [delegateToSchema](https://www.graphql-tools.com/docs/schema-delegation/#delegatetoschema) documentation for more details on available `options` since the delegateToComponent fuctions is simply an adapter for the `GraphQLComponent` API.

A GraphQLComponent instance (ie, `new GraphQLComponent({...})`) has the following API:

- `schema` - getter that this component's `GraphQLSchema` object (ie. the "executable" schema that is constructed as described above)
- `context` - context function that builds context for all components in the tree.
- `types` - this component's types.
- `resolvers` - this component's resolvers.
- `imports` - this component's imported components in the form of import configuration objects
- `mocks` - custom mocks for this component.
- `directives` - this component's directives.
- `dataSources` - this component's data source(s), if any.

# General usage

Creating a component using the GraphQLComponent class:

```javascript
const GraphQLComponent = require('graphql-component');

const { schema, context } = new GraphQLComponent({ types, resolvers });
```

### Encapsulating state

Typically the best way to make a re-useable component with instance data will be to extend `GraphQLComponent`.

```javascript
const GraphQLComponent = require('graphql-component');
const resolvers = require('./resolvers');
const types = require('./types');
const mocks = require('./mocks');

class PropertyComponent extends GraphQLComponent {
  constructor({ types, resolvers }) {
    super({ types, resolvers });
  }
}

module.exports = PropertyComponent;
```

### Aggregation

Example to merge multiple components:

```javascript
const { schema, context } = new GraphQLComponent({
  imports: [
    new Property(),
    new Reviews()
  ]
});

const server = new ApolloServer({
  schema,
  context
});
```

### Import configuration

Imports can be a configuration object supplying the following properties:

- `component` - the component instance to import.
- `exclude` - fields, if any, to exclude.

### Exclude

You can exclude whole types or individual fields on types.

```javascript
const { schema, context } = new GraphQLComponent({
  imports: [
    {
      component: new Property(),
      exclude: ['Mutation.*']
    },
    {
      component: new Reviews(),
      exclude: ['Mutation.*']
    }
  ]
});
```

The excluded types will not appear in the aggregate or gateway schema exposed by the root component, but are still present in the schema encapsulated by the underlying component. This can keep from leaking unintended surface area, but you can still delegate calls to imported component's schema to utilize the excluded field under the covers.

### Data Source support

Data sources in `graphql-component` do not extend `apollo-datasource`'s `DataSource` class.

Instead, data sources in components will be injected into the context, but wrapped in a proxy such that the global
context will be injected as the first argument of any function implemented in a data source class.

This allows there to exist one instance of a data source for caching or other statefullness (like circuit breakers),
while still ensuring that a data source will have the current context.

For example, a data source should be implemented like:

```javascript
class PropertyDataSource {
  async getPropertyById(context, id) {
    //do some work...
  }
}
```

This data source would be executed without passing the `context` manually:

```javascript
const resolvers = {
  Query: {
    property(_, { id }, { dataSources }) {
      return dataSources.PropertyDataSource.getPropertyById(id);
    }
  }
}
```

Setting up a component to use a data source might look like:

```javascript
new GraphQLComponent({
  //...
  dataSources: [new PropertyDataSource()]
})
```

### Override data sources

Since data sources are added to the context based on the constructor name, it is possible to simply override data sources by passing the same class name or overriding the constructor name:

```javascript
const { schema, context } = new GraphQLComponent({
  imports: [
    {
      component: new Property(),
      exclude: ['Mutation.*']
    },
    {
      component: new Reviews(),
      exclude: ['Mutation.*']
    }
  ],
  dataSourceOverrides: [
    new class PropertyMock {
      static get name() {
        return 'PropertyDataSource';
      }
      //...etc
    }
  ]
});
```

### Decorating the global context

Example context argument:

```javascript
const context = {
  namespace: 'myNamespace',
  factory: function ({ req }) {
    return 'my value';
  }
};
```

After this, resolver `context` will contain `{ ..., myNamespace: 'my value' }`.

### Context middleware

It may be necessary to transform the context before invoking component context.

```javascript
const { schema, context } = new GraphQLComponent({types, resolvers, context});

context.use('transformRawRequest', ({ request }) => {
  return { req: request.raw.req };
});
```

Using `context` now in `apollo-server-hapi` for example, will transform the context to one similar to default `apollo-server`.