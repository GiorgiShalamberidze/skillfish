---
name: graphql-architect
description: GraphQL schema design, resolvers, federation, subscriptions, DataLoader patterns, caching strategies, and security with query cost analysis.
---

# GraphQL Architect

## Overview

The GraphQL Architect skill provides deep, production-grade knowledge for designing, building, and operating GraphQL APIs at scale. It covers the full lifecycle from schema-first design through federation, real-time subscriptions, caching, security hardening, and performance optimization. Whether you are building a greenfield API or migrating a monolith REST surface to a federated graph, this skill gives you the patterns, code, and decision frameworks to ship with confidence.

## Table of Contents

1. [Schema Design](#1-schema-design)
2. [Resolver Patterns](#2-resolver-patterns)
3. [Federation](#3-federation)
4. [Subscriptions](#4-subscriptions)
5. [Caching](#5-caching)
6. [Security](#6-security)
7. [Performance](#7-performance)
8. [Tooling Quick Reference](#8-tooling-quick-reference)

---

## 1. Schema Design

### SDL Fundamentals

Write schemas in SDL (Schema Definition Language) first, then generate types. Schema-first keeps the contract readable by frontend and backend teams alike.

```graphql
"""
A registered user of the platform.
"""
type User {
  id: ID!
  email: String!
  displayName: String!
  avatar: URL
  role: Role!
  posts(first: Int = 10, after: String): PostConnection!
  createdAt: DateTime!
}
```

**Conventions:**
- Document every type and field with triple-quote descriptions
- Use `ID!` for primary identifiers, never `Int!`
- Default pagination arguments in the field signature
- Keep root Query/Mutation/Subscription types thin -- delegate to domain types

### Type Design Principles

- **Prefer specific types over generic ones.** A `Money` type with `amount` and `currency` beats a raw `Float`.
- **One responsibility per type.** If a type serves two unrelated consumers, split it.
- **Avoid "god" input types.** Create separate inputs for create vs update mutations.

```graphql
type Money {
  amount: Int!          # cents / smallest unit
  currency: Currency!
}

enum Currency {
  USD
  EUR
  GBP
  JPY
}
```

### Nullability Strategy

Nullability communicates reliability guarantees to clients.

| Guideline | Example | Rationale |
|---|---|---|
| Required fields are `!` | `email: String!` | Clients never need null checks |
| Lists are non-null with non-null items | `tags: [String!]!` | Prevents `null` list or `null` items |
| Fields backed by unreliable sources are nullable | `weatherForecast: Forecast` | Allows partial responses on failure |
| Mutation return types are non-null | `createUser: CreateUserPayload!` | Always return a typed payload |

### Input Types

Separate inputs for create and update prevent accidental overwrites and keep validation explicit.

```graphql
input CreatePostInput {
  title: String!
  body: String!
  categoryId: ID!
  tags: [String!]
  publishAt: DateTime
}

input UpdatePostInput {
  title: String
  body: String
  categoryId: ID
  tags: [String!]
  publishAt: DateTime
}
```

**Best practices:**
- Never reuse an output type as an input
- Prefix inputs with the mutation name: `CreatePostInput`, `UpdatePostInput`
- Add a `clientMutationId: String` field when clients need optimistic UI correlation

### Enums

Use enums for any field with a fixed set of values. They provide compile-time safety and self-documenting schemas.

```graphql
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum SortDirection {
  ASC
  DESC
}
```

**Guidelines:**
- Use UPPER_SNAKE_CASE for enum values
- Add descriptions for non-obvious values
- Avoid enums that change frequently -- consider a `String` with validation instead
- Expose enums for filter arguments so clients can discover valid options

### Interfaces vs Unions

Use **interfaces** when types share fields; use **unions** when types are conceptually related but structurally different.

```graphql
# Interface -- shared fields, each implementor adds its own
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type User implements Node & Timestamped {
  id: ID!
  email: String!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Organization implements Node & Timestamped {
  id: ID!
  name: String!
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Union -- no shared fields, exhaustive fragment matching
union SearchResult = User | Post | Comment | Organization

type Query {
  search(query: String!): [SearchResult!]!
  node(id: ID!): Node
}
```

**Decision tree:**
1. Do the types share fields? --> Interface
2. Are they returned from the same field but structurally different? --> Union
3. Need both? --> Interface with a union wrapping specific implementors

### Custom Scalars

Define custom scalars for domain-specific data to get schema-level validation and documentation.

```graphql
scalar DateTime    # ISO-8601: 2026-04-04T12:00:00Z
scalar URL         # RFC 3986 compliant URI
scalar EmailAddress
scalar JSON        # Escape hatch -- use sparingly
scalar PositiveInt
scalar UUID
```

```typescript
import { GraphQLScalarType, Kind } from 'graphql';

export const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO-8601 date-time string',
  serialize(value: Date): string {
    return value.toISOString();
  },
  parseValue(value: string): Date {
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new TypeError(`Invalid DateTime: ${value}`);
    }
    return date;
  },
  parseLiteral(ast): Date {
    if (ast.kind === Kind.STRING) {
      return new Date(ast.value);
    }
    throw new TypeError('DateTime must be a string');
  },
});
```

---

## 2. Resolver Patterns

### Field Resolvers

Every field in the schema maps to a resolver. Keep resolvers thin -- they orchestrate, they do not contain business logic.

```typescript
const resolvers: Resolvers = {
  Query: {
    user: (_parent, { id }, ctx) => ctx.dataSources.userService.getById(id),
    users: (_parent, { first, after }, ctx) =>
      ctx.dataSources.userService.paginate({ first, after }),
  },
  User: {
    posts: (user, { first, after }, ctx) =>
      ctx.dataSources.postService.getByAuthor(user.id, { first, after }),
    fullName: (user) => `${user.firstName} ${user.lastName}`,
  },
};
```

### Context Object

Build context once per request. It carries authentication, data sources, and request-scoped state.

```typescript
interface GraphQLContext {
  currentUser: User | null;
  dataSources: {
    userService: UserService;
    postService: PostService;
    commentService: CommentService;
  };
  loaders: {
    userLoader: DataLoader<string, User>;
    postLoader: DataLoader<string, Post>;
  };
  requestId: string;
  logger: Logger;
}

async function buildContext({ req }: { req: Request }): Promise<GraphQLContext> {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const currentUser = token ? await verifyToken(token) : null;
  const requestId = crypto.randomUUID();

  return {
    currentUser,
    dataSources: {
      userService: new UserService(db),
      postService: new PostService(db),
      commentService: new CommentService(db),
    },
    loaders: createLoaders(db),
    requestId,
    logger: rootLogger.child({ requestId }),
  };
}
```

### DataLoader for N+1

DataLoader batches and caches individual loads within a single request tick, eliminating N+1 queries.

```typescript
import DataLoader from 'dataloader';

function createLoaders(db: Database) {
  return {
    userLoader: new DataLoader<string, User>(async (ids) => {
      const users = await db.query(
        'SELECT * FROM users WHERE id = ANY($1)',
        [ids]
      );
      // DataLoader requires results in the same order as input ids
      const userMap = new Map(users.map((u) => [u.id, u]));
      return ids.map((id) => userMap.get(id) ?? new Error(`User ${id} not found`));
    }),

    postLoader: new DataLoader<string, Post>(async (ids) => {
      const posts = await db.query(
        'SELECT * FROM posts WHERE id = ANY($1)',
        [ids]
      );
      const postMap = new Map(posts.map((p) => [p.id, p]));
      return ids.map((id) => postMap.get(id) ?? new Error(`Post ${id} not found`));
    }),

    // 1:many loader -- key is the foreign key, returns arrays
    postsByAuthorLoader: new DataLoader<string, Post[]>(async (authorIds) => {
      const posts = await db.query(
        'SELECT * FROM posts WHERE author_id = ANY($1)',
        [authorIds]
      );
      const grouped = new Map<string, Post[]>();
      for (const post of posts) {
        const list = grouped.get(post.authorId) ?? [];
        list.push(post);
        grouped.set(post.authorId, list);
      }
      return authorIds.map((id) => grouped.get(id) ?? []);
    }),
  };
}
```

**Key rules:**
- Create loaders per-request (in `buildContext`), never as singletons
- Always return results in the same order as input keys
- Return `Error` instances for missing items, not `null` (lets DataLoader distinguish)
- Use `{ cache: false }` if the same key may return different results within one request

### Batch Loading Strategies

| Strategy | When to Use | Example |
|---|---|---|
| Simple batch | Load N items by ID | `SELECT * FROM users WHERE id IN (...)` |
| Grouped batch | Load related items by FK | `SELECT * FROM posts WHERE author_id IN (...)` |
| Composite key | Multi-column lookup | `DataLoader<{orgId, userId}, Membership>` |
| Primed loader | Avoid re-fetching known data | `loader.prime(id, existingObject)` |

### Resolver Composition

Use higher-order functions for cross-cutting concerns instead of duplicating logic.

```typescript
// Auth wrapper
function requireAuth<TParent, TArgs, TResult>(
  resolver: (parent: TParent, args: TArgs, ctx: GraphQLContext) => TResult
) {
  return (parent: TParent, args: TArgs, ctx: GraphQLContext) => {
    if (!ctx.currentUser) {
      throw new AuthenticationError('Authentication required');
    }
    return resolver(parent, args, ctx);
  };
}

// Role wrapper
function requireRole<TParent, TArgs, TResult>(
  role: Role,
  resolver: (parent: TParent, args: TArgs, ctx: GraphQLContext) => TResult
) {
  return requireAuth((parent: TParent, args: TArgs, ctx: GraphQLContext) => {
    if (ctx.currentUser!.role !== role) {
      throw new ForbiddenError(`Requires role: ${role}`);
    }
    return resolver(parent, args, ctx);
  });
}

// Usage
const resolvers = {
  Mutation: {
    deleteUser: requireRole(Role.ADMIN, (_p, { id }, ctx) =>
      ctx.dataSources.userService.delete(id)
    ),
    updateProfile: requireAuth((_p, { input }, ctx) =>
      ctx.dataSources.userService.update(ctx.currentUser!.id, input)
    ),
  },
};
```

### Mutation Payloads

Always return typed payloads from mutations, never raw types.

```graphql
type CreateUserPayload {
  user: User
  errors: [UserError!]!
}

type UserError {
  field: String!
  message: String!
  code: ErrorCode!
}

enum ErrorCode {
  VALIDATION_FAILED
  NOT_FOUND
  DUPLICATE
  FORBIDDEN
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}
```

```typescript
const resolvers = {
  Mutation: {
    createUser: async (_p, { input }, ctx): Promise<CreateUserPayload> => {
      const errors = validateCreateUser(input);
      if (errors.length > 0) {
        return { user: null, errors };
      }
      const user = await ctx.dataSources.userService.create(input);
      return { user, errors: [] };
    },
  },
};
```

---

## 3. Federation

### When to Federate

Federate when you have multiple teams owning distinct domains, or when your monolith schema exceeds what one team can maintain. Do not federate a single-team API -- the overhead is not worth it.

### Subgraph Design

Each subgraph owns a bounded context. It defines the types it is authoritative for and extends types owned by other subgraphs.

```graphql
# -- users subgraph --
type User @key(fields: "id") {
  id: ID!
  email: String!
  displayName: String!
  role: Role!
}

type Query {
  me: User
  user(id: ID!): User
}
```

```graphql
# -- posts subgraph --
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  author: User!
  createdAt: DateTime!
}

# Extend User from the users subgraph
extend type User @key(fields: "id") {
  id: ID! @external
  posts(first: Int = 10, after: String): PostConnection!
}

type Query {
  post(id: ID!): Post
  feed(first: Int = 20, after: String): PostConnection!
}
```

### Federation Directives

| Directive | Purpose | Example |
|---|---|---|
| `@key` | Declares an entity's primary key for cross-subgraph resolution | `@key(fields: "id")` |
| `@external` | Marks a field as owned by another subgraph | `id: ID! @external` |
| `@requires` | Field needs external fields to resolve | `@requires(fields: "weight dimension")` |
| `@provides` | Resolver provides additional fields on a return type | `@provides(fields: "name")` |
| `@shareable` | Multiple subgraphs can resolve this field | `type Product @shareable` |
| `@inaccessible` | Hide a field from the composed supergraph | `internalNote: String @inaccessible` |
| `@override` | Migrate a field from one subgraph to another | `@override(from: "legacy")` |

### Entity Resolution

Each subgraph must provide a reference resolver so the gateway can stitch entities together.

```typescript
// users subgraph
const resolvers = {
  User: {
    __resolveReference: async (ref: { id: string }, ctx) => {
      return ctx.dataSources.userService.getById(ref.id);
    },
  },
};
```

```typescript
// posts subgraph -- resolves User.posts
const resolvers = {
  User: {
    posts: async (user: { id: string }, { first, after }, ctx) => {
      return ctx.dataSources.postService.getByAuthor(user.id, { first, after });
    },
  },
};
```

### Gateway Configuration

```typescript
import { ApolloGateway, IntrospectAndCompose } from '@apollo/gateway';
import { ApolloServer } from '@apollo/server';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://users-service:4001/graphql' },
      { name: 'posts', url: 'http://posts-service:4002/graphql' },
      { name: 'comments', url: 'http://comments-service:4003/graphql' },
    ],
  }),
});

const server = new ApolloServer({ gateway });
```

For production, prefer managed federation with a schema registry (Apollo GraphOS, Hive) over `IntrospectAndCompose`.

### Migration from Monolith Schema

**Phase 1 -- Strangler Fig:**
1. Deploy the gateway in front of the monolith subgraph
2. All traffic routes through the gateway; no client changes needed

**Phase 2 -- Extract First Subgraph:**
1. Pick the least-coupled domain (e.g. notifications)
2. Create a new subgraph with its own types
3. Use `@override` to migrate fields one at a time
4. Run both subgraphs behind the gateway; validate with schema checks

**Phase 3 -- Iterate:**
1. Extract the next domain, repeat
2. Shrink the monolith subgraph progressively
3. Remove `@override` directives once the monolith no longer serves the migrated fields

---

## 4. Subscriptions

### WebSocket Transport

GraphQL subscriptions use a persistent WebSocket connection. The standard protocol is `graphql-ws` (not the legacy `subscriptions-transport-ws`).

```typescript
import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';

const app = express();
const httpServer = createServer(app);

const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql',
});

const schema = makeExecutableSchema({ typeDefs, resolvers });

useServer(
  {
    schema,
    context: async (ctx) => {
      const token = ctx.connectionParams?.authToken as string;
      const currentUser = token ? await verifyToken(token) : null;
      return { currentUser };
    },
    onConnect: async (ctx) => {
      const token = ctx.connectionParams?.authToken as string;
      if (!token) return false; // reject unauthenticated connections
    },
  },
  wsServer
);

const server = new ApolloServer({ schema });
await server.start();
app.use('/graphql', express.json(), expressMiddleware(server));

httpServer.listen(4000);
```

### PubSub Patterns

Use an external PubSub for multi-instance deployments. In-memory PubSub is only suitable for single-process development.

```typescript
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

const pubsub = new RedisPubSub({
  publisher: new Redis(process.env.REDIS_URL),
  subscriber: new Redis(process.env.REDIS_URL),
});

// Schema
const typeDefs = `
  type Subscription {
    postCreated: Post!
    commentAdded(postId: ID!): Comment!
    orderStatusChanged(orderId: ID!): Order!
  }
`;

// Publishing from a mutation
const resolvers = {
  Mutation: {
    createPost: async (_p, { input }, ctx) => {
      const post = await ctx.dataSources.postService.create(input);
      await pubsub.publish('POST_CREATED', { postCreated: post });
      return { post, errors: [] };
    },
  },
  Subscription: {
    postCreated: {
      subscribe: () => pubsub.asyncIterableIterator(['POST_CREATED']),
    },
  },
};
```

### Filtering

Filter events server-side so clients only receive relevant updates.

```typescript
import { withFilter } from 'graphql-subscriptions';

const resolvers = {
  Subscription: {
    commentAdded: {
      subscribe: withFilter(
        () => pubsub.asyncIterableIterator(['COMMENT_ADDED']),
        (payload, variables) => {
          return payload.commentAdded.postId === variables.postId;
        }
      ),
    },
    orderStatusChanged: {
      subscribe: withFilter(
        () => pubsub.asyncIterableIterator(['ORDER_STATUS_CHANGED']),
        (payload, variables, ctx) => {
          // Only send to the user who owns the order
          return (
            payload.orderStatusChanged.id === variables.orderId &&
            payload.orderStatusChanged.userId === ctx.currentUser?.id
          );
        }
      ),
    },
  },
};
```

### Scaling Subscriptions

| Challenge | Solution |
|---|---|
| Multiple server instances | External PubSub (Redis, Kafka, NATS) |
| Connection limits | Sticky sessions or dedicated subscription tier |
| Memory pressure | Cap subscriptions per user; disconnect idle connections |
| Gateway routing | Use `@apollo/gateway` subscription support or a dedicated WebSocket gateway |
| Heartbeats / keepalive | `graphql-ws` handles ping/pong automatically; configure interval |

---

## 5. Caching

### Response Caching

Cache full query responses when the same query with the same variables is repeated frequently.

```typescript
import responseCachePlugin from '@apollo/server-plugin-response-cache';

const server = new ApolloServer({
  schema,
  plugins: [
    responseCachePlugin({
      sessionId: (ctx) => ctx.request.http?.headers.get('authorization') ?? null,
    }),
  ],
});
```

### Persisted Queries & Automatic Persisted Queries (APQ)

Persisted queries replace full query strings with hashes, reducing payload size and preventing arbitrary query execution.

**APQ flow:**
1. Client sends `{ extensions: { persistedQuery: { sha256Hash: "abc123" } } }`
2. Server checks cache -- if found, executes; if not, returns `PersistedQueryNotFound`
3. Client retries with full query body + hash
4. Server caches the mapping for future requests

```typescript
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginCacheControl } from '@apollo/server/plugin/cacheControl';
import { KeyvAdapter } from '@apollo/utils.keyvadapter';
import Keyv from 'keyv';

const server = new ApolloServer({
  schema,
  cache: new KeyvAdapter(new Keyv('redis://localhost:6379')),
  plugins: [
    ApolloServerPluginCacheControl({ defaultMaxAge: 60 }),
  ],
  persistedQueries: {
    cache: new KeyvAdapter(new Keyv('redis://localhost:6379')),
    ttl: 900, // 15 minutes
  },
});
```

### @cacheControl Directive

Set cache hints at the schema level. The gateway/CDN uses these to set `Cache-Control` headers.

```graphql
type Query {
  popularPosts: [Post!]! @cacheControl(maxAge: 300)       # 5 min
  currentUser: User @cacheControl(maxAge: 0, scope: PRIVATE)
}

type Post @cacheControl(maxAge: 120) {
  id: ID!
  title: String!
  body: String!
  viewCount: Int! @cacheControl(maxAge: 30)               # shorter for volatile field
  author: User!
}
```

**Cache scopes:**
- `PUBLIC` -- response can be cached by CDNs and shared caches
- `PRIVATE` -- response is user-specific and must not be shared

### CDN Caching

Place a CDN (Cloudflare, Fastly, CloudFront) in front of the GraphQL endpoint for `GET` requests with persisted queries.

```
Client --> CDN --> GraphQL Server
         (cache hit? serve directly)
```

**Requirements for CDN caching:**
- Use `GET` requests (query string or persisted query hash)
- Set proper `Cache-Control` headers via `@cacheControl`
- Vary on `Authorization` header for private data
- Purge CDN cache on mutations that affect cached data

### Client-Side Normalized Cache

Apollo Client, urql, and Relay all maintain normalized caches keyed by `__typename` + `id`.

```typescript
import { ApolloClient, InMemoryCache } from '@apollo/client';

const client = new ApolloClient({
  uri: '/graphql',
  cache: new InMemoryCache({
    typePolicies: {
      Post: {
        fields: {
          // Merge paginated results
          comments: {
            keyArgs: false,
            merge(existing = { edges: [] }, incoming) {
              return {
                ...incoming,
                edges: [...existing.edges, ...incoming.edges],
              };
            },
          },
        },
      },
      Query: {
        fields: {
          feed: {
            keyArgs: ['filter'],
            merge(existing, incoming, { args }) {
              if (!args?.after) return incoming;
              return {
                ...incoming,
                edges: [...(existing?.edges ?? []), ...incoming.edges],
              };
            },
          },
        },
      },
    },
  }),
});
```

---

## 6. Security

### Depth Limiting

Prevent deeply nested queries that consume excessive server resources.

```typescript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  schema,
  validationRules: [depthLimit(10)],
});
```

**Guideline:** Set the limit to 2-3 levels beyond your deepest legitimate query. Audit client queries before deploying.

### Query Cost Analysis

Assign a cost to each field and reject queries that exceed a budget. This is more nuanced than depth limiting alone.

```typescript
import { createComplexityRule, simpleEstimator, fieldExtensionsEstimator } from 'graphql-query-complexity';

const complexityRule = createComplexityRule({
  maximumComplexity: 1000,
  estimators: [
    fieldExtensionsEstimator(),
    simpleEstimator({ defaultComplexity: 1 }),
  ],
  onComplete: (complexity) => {
    console.log('Query complexity:', complexity);
  },
});

const server = new ApolloServer({
  schema,
  validationRules: [complexityRule],
});
```

Annotate expensive fields in the schema or resolver map:

```graphql
type Query {
  search(query: String!, first: Int = 20): SearchConnection!   # cost: first * 5
  analytics(range: DateRange!): AnalyticsReport!               # cost: 50
}
```

```typescript
const resolvers = {
  Query: {
    search: {
      resolve: (_p, args, ctx) => ctx.dataSources.searchService.query(args),
      extensions: {
        complexity: ({ args }) => (args.first ?? 20) * 5,
      },
    },
    analytics: {
      resolve: (_p, args, ctx) => ctx.dataSources.analyticsService.report(args),
      extensions: {
        complexity: () => 50,
      },
    },
  },
};
```

### Rate Limiting by Complexity

Combine query cost with a sliding window rate limiter so expensive queries consume more budget.

```typescript
import { RateLimiterRedis } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterRedis({
  storeClient: redisClient,
  keyPrefix: 'gql_rate',
  points: 5000,       // max complexity points
  duration: 60,        // per 60 seconds
});

async function rateLimitPlugin() {
  return {
    async requestDidStart() {
      return {
        async didResolveOperation(ctx) {
          const complexity = getQueryComplexity(ctx);
          const key = ctx.contextValue.currentUser?.id ?? ctx.contextValue.ip;
          try {
            await rateLimiter.consume(key, complexity);
          } catch {
            throw new Error('Rate limit exceeded. Reduce query complexity or wait.');
          }
        },
      };
    },
  };
}
```

### Field-Level Authorization

Use schema directives or resolver wrappers for fine-grained access control.

```graphql
directive @auth(requires: Role = USER) on FIELD_DEFINITION | OBJECT

type User @auth(requires: USER) {
  id: ID!
  email: String! @auth(requires: ADMIN)
  displayName: String!
  role: Role! @auth(requires: ADMIN)
  posts: PostConnection!
}
```

```typescript
import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';

function authDirectiveTransformer(schema: GraphQLSchema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const authDirective = getDirective(schema, fieldConfig, 'auth')?.[0];
      if (!authDirective) return fieldConfig;

      const requiredRole = authDirective.requires;
      const originalResolve = fieldConfig.resolve ?? defaultFieldResolver;

      fieldConfig.resolve = async (parent, args, ctx, info) => {
        if (!ctx.currentUser) {
          throw new AuthenticationError('Not authenticated');
        }
        if (!hasRole(ctx.currentUser, requiredRole)) {
          throw new ForbiddenError(`Requires role: ${requiredRole}`);
        }
        return originalResolve(parent, args, ctx, info);
      };

      return fieldConfig;
    },
  });
}
```

### Introspection Controls

Disable introspection in production to prevent schema discovery by attackers.

```typescript
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginInlineTraceDisabled } from '@apollo/server/plugin/disabled';

const server = new ApolloServer({
  schema,
  introspection: process.env.NODE_ENV !== 'production',
  plugins: [
    // Disable inline tracing in production
    ...(process.env.NODE_ENV === 'production'
      ? [ApolloServerPluginInlineTraceDisabled()]
      : []),
  ],
});
```

**Additional hardening:**
- Strip `__schema` and `__type` queries with a validation rule in production
- Use an allowlist of persisted queries -- block all ad-hoc queries
- Mask error messages: never leak stack traces or internal details

```typescript
const server = new ApolloServer({
  schema,
  formatError: (formattedError, error) => {
    // Log full error internally
    logger.error(error);
    // Return safe message to client
    if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
      return { message: 'An unexpected error occurred', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
    }
    return formattedError;
  },
});
```

---

## 7. Performance

### Query Planning

The gateway builds a query plan to determine which subgraphs to call and in what order. Understanding query plans helps you optimize schema design.

```
QueryPlan {
  Sequence {
    Fetch(service: "users") {
      { user(id: "1") { id displayName } }
    }
    Flatten(path: "user") {
      Fetch(service: "posts") {
        { ... on User { posts { edges { node { id title } } } } }
      }
    }
  }
}
```

**Optimization tips:**
- Co-locate fields that are commonly queried together in the same subgraph
- Minimize cross-subgraph `@requires` -- each one adds a fetch hop
- Use `@provides` to prefetch fields when you know they will be needed

### @defer and @stream

Incremental delivery lets the server send partial results as they become available, improving perceived latency.

```graphql
# @defer -- delay expensive section
query PostWithComments($id: ID!) {
  post(id: $id) {
    id
    title
    body
    ... @defer {
      comments(first: 50) {
        edges {
          node {
            id
            body
            author { displayName }
          }
        }
      }
    }
  }
}

# @stream -- stream list items one at a time
query Feed {
  feed(first: 20) {
    edges @stream(initialCount: 5) {
      node {
        id
        title
        summary
      }
    }
  }
}
```

**Requirements:**
- Server must support incremental delivery (Apollo Server 4+, GraphQL Yoga)
- Client must handle multipart responses
- Use `@defer` for heavy nested data; use `@stream` for long lists

### Pagination: Cursor vs Offset

| Aspect | Cursor (Relay-style) | Offset |
|---|---|---|
| Stability | Stable across inserts/deletes | Shifts when data changes |
| Performance | O(1) seek with indexed cursor | O(n) skip for large offsets |
| Jump to page | Not supported natively | Supported |
| Implementation | More complex | Simpler |
| Recommended for | Infinite scroll, real-time feeds | Admin tables with page numbers |

**Cursor pagination (Relay spec):**

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}
```

```typescript
async function cursorPaginate(
  table: string,
  { first = 20, after }: { first?: number; after?: string }
): Promise<Connection> {
  const limit = Math.min(first, 100); // cap page size
  let query = `SELECT * FROM ${table}`;
  const params: unknown[] = [];

  if (after) {
    const cursor = decodeCursor(after); // base64 -> { id, createdAt }
    query += ` WHERE (created_at, id) < ($1, $2)`;
    params.push(cursor.createdAt, cursor.id);
  }

  query += ` ORDER BY created_at DESC, id DESC LIMIT $${params.length + 1}`;
  params.push(limit + 1); // fetch one extra to determine hasNextPage

  const rows = await db.query(query, params);
  const hasNextPage = rows.length > limit;
  const edges = rows.slice(0, limit).map((row) => ({
    cursor: encodeCursor({ id: row.id, createdAt: row.created_at }),
    node: row,
  }));

  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!after,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    totalCount: await countRows(table),
  };
}
```

### Query Batching

Batch multiple queries in a single HTTP request to reduce round trips.

```typescript
// Client sends an array of operations
const batchedRequest = [
  { query: '{ me { id displayName } }' },
  { query: '{ notifications { unreadCount } }' },
  { query: '{ feed(first: 5) { edges { node { title } } } }' },
];

// Server config -- enable batching with limits
const server = new ApolloServer({
  schema,
  allowBatchedHttpRequests: true,
});
```

**Guard rails for batching:**
- Limit batch size (e.g., max 10 operations per request)
- Sum complexity across all operations in the batch
- Apply rate limiting to the batch as a whole

---

## 8. Tooling Quick Reference

### Schema & Development

| Tool | Purpose |
|---|---|
| [GraphQL Code Generator](https://the-guild.dev/graphql/codegen) | Generate TypeScript types, resolvers, hooks from schema |
| [GraphQL Yoga](https://the-guild.dev/graphql/yoga-server) | Lightweight, spec-compliant server (Envelop plugin system) |
| [Apollo Server](https://www.apollographql.com/docs/apollo-server/) | Full-featured server with plugin ecosystem |
| [Pothos](https://pothos-graphql.dev/) | Code-first schema builder for TypeScript |
| [Nexus](https://nexusjs.org/) | Code-first schema builder (alternative to Pothos) |
| [GraphQL Mesh](https://the-guild.dev/graphql/mesh) | Unify REST, gRPC, SOAP, and other sources as GraphQL |

### Federation & Gateway

| Tool | Purpose |
|---|---|
| [Apollo Router](https://www.apollographql.com/docs/router/) | High-performance Rust gateway for federated graphs |
| [Apollo GraphOS](https://www.apollographql.com/graphos) | Managed schema registry, checks, analytics |
| [GraphQL Hive](https://the-guild.dev/graphql/hive) | Open-source schema registry and analytics |
| [Cosmo Router](https://wundergraph.com/cosmo) | Open-source federation runtime (WunderGraph) |

### Testing & Validation

| Tool | Purpose |
|---|---|
| [GraphQL Inspector](https://the-guild.dev/graphql/inspector) | Schema diff, breaking change detection, coverage |
| [graphql-eslint](https://the-guild.dev/graphql/eslint) | ESLint rules for GraphQL SDL and operations |
| [Apollo Sandbox](https://studio.apollographql.com/sandbox) | Browser IDE for exploring and testing queries |
| [Stellate](https://stellate.co/) | Edge caching and analytics for GraphQL APIs |

### Client Libraries

| Tool | Purpose |
|---|---|
| [Apollo Client](https://www.apollographql.com/docs/react/) | Full-featured client with normalized cache (React, iOS, Kotlin) |
| [urql](https://commerce.nearform.com/open-source/urql/) | Lightweight, extensible client (React, Vue, Svelte) |
| [Relay](https://relay.dev/) | Meta's client, optimized for large apps with compiler |
| [gql.tada](https://gql-tada.0no.co/) | TypeScript-first client with inferred types from schema |
| [Genql](https://genql.dev/) | Type-safe, auto-complete GraphQL client generator |

### Performance & Monitoring

| Tool | Purpose |
|---|---|
| [DataLoader](https://github.com/graphql/dataloader) | Batch and cache per-request data loads |
| [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) | Query cost analysis and limits |
| [graphql-depth-limit](https://github.com/stems/graphql-depth-limit) | Reject deeply nested queries |
| [Stellate](https://stellate.co/) | Edge caching, rate limiting, analytics |
| [Apollo Studio](https://studio.apollographql.com/) | Trace-level performance monitoring, field usage |
