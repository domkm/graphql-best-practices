# GraphQL Best Practices

This document makes the following assumptions. If these assumptions are not true for your use-case then these recommendations may not apply.

- The protocol is HTTP and/or WebSocket
- The data format is JSON
- Your clients are GUIs
- Your clients are internal (in other words, your API is not public like GitHub or Shopify)

Consider reading the [spec](https://graphql.github.io/graphql-spec/). It is quite short.

It may also help to read the official [GraphQL Best Practices](https://graphql.org/learn/best-practices/).

This guide assumes knowledge of the GraphQL spec. This document is about the art of using GraphQL effectively. As such, virtually everything here is subjective.

- [The Basics](#the-basics)
  - [JSON](#json)
  - [Headers](#headers)
  - [Status Codes](#status-codes)
  - [Authentication](#authentication)
- [Schema](#schema)
  - [Names](#names)
  - [Documentation](#documentation)
  - [Custom Scalars](#custom-scalars)
  - [Field Names and Types](#field-names-and-types)
  - [Interfaces and Unions](#interfaces-and-unions)
  - [References](#references)
  - [Relay](#relay)
  - [Current User](#current-user)
  - [Arguments](#arguments)
  - [Field Nullability](#field-nullability)
  - [Client-Driven](#client-driven)
  - [Redundant Fields](#redundant-fields)
  - [Mutations](#mutations)
  - [Files](#files)
- [Server](#server)
  - [Schema Creation](#schema-creation)
  - [Authorization](#authorization)
  - [Deprecation](#deprecation)
- [Client](#client)
  - [Deprecation](#deprecation-1)
  - [Ecosystem](#ecosystem)
  - [Static Queries](#static-queries)
  - [Schema Type System](#schema-type-system)
  - [Local State Management](#local-state-management)
- [Optimization](#optimization)
  - [Persisted and Whitelisted Queries](#persisted-and-whitelisted-queries)

## The Basics

Since we're using HTTP, there are multiple ways for the client to send information to the server. We could use headers, URLs, and POST bodies.

### JSON

The GraphQL spec is oddly silent about the format of requests. However, all tools and libraries that I am aware of use the following format:

```json
{
  "query": "[GraphQL document string]",
  "variables": {
    "optional": "map of variables"
  },
  "operationName": "RequiredIfDocumentIncludesMultipleOperations"
}
```

Generally, `operationName` is omitted. However, when sending a document with multiple queries, `operationName` is used to specify which query to run.

```graphql
# Only `Query1` or `Query2` can be run in a single request,
# so `operationName: "Query2"` could be used to specify which to run.
query Query1 {
  # ...
}
query Query2 {
  # ...
}
```

### Headers

Headers should be used for all information that is expected to be available in all resolvers. For example, a token to authenticate the current user might be sent in the `Authorization` header or the current user's language preference might be sent in the `Accept-Language` header. These are not specific to any field and should, therefore, be sent via headers. Do _not_ try to do something like this:

```graphql
query {
  request(authToken: "...", language: "en-US") {
    # ...
  }
}
```
This leads to very complicated queries and context-dependent resolvers.

### Status Codes

Servers should always send 200 status codes. Errors should be conveyed in response `errors` array.

### Authentication

Authentication can be done via mutations (like `authenticateUser`) or externally (directly through REST calls or via a provider like Auth0). Cookie-based authentication is fine, though JWTs are far more common. Either way, use headers to send tokens; do not put those in GraphQL requests.

## Schema

### Names

#### Cases

- Type names should `PascalCase`
- Field and argument names should be `camelCase`
- Enum values should be `SCREAMING_SNAKE_CASE`.

#### Types

Name the root types `Query`, `Mutation`, and `Subscription`.

Name the current user type `Viewer` or `Me`.

Name mutation input objects and output objects `[NameOfMutation]Input` and `[NameOfMutation]Payload` respectively.

#### List Fields

Fields that return lists or collection wrappers (like Relay connections) should be plural.

```graphql
# good
type User {
  memberships: [Membership!]
}

# bad
type User {
  membership: [Membership!]
}
```

### Documentation

Use GraphQL descriptions everywhere: types, fields, arguments, etc.

Do not make front-end developers guess the meaning or functionality of the schema; tell them.

### Custom Scalars

Use custom scalars (in moderation).

Custom scalars allow schemas to be extended with new scalars that are outside of the GraphQL spec. Use this feature, but only for well-understood domains that offer significant leverage.

Good examples:
- `ISO8601DateTime`
- `E164PhoneNumber`
- `EmailAddress`
- `URL`
- `JSON`

Bad examples:
- `Name` - just use String
- `Char32` - just use String

### Field Names and Types

Prefer very specific field names or nested object types.

Use very descriptive and specific names, especially while the domain is not yet well-understood. This allows the use of more general names when the domain is more understood. Typically, general names are best reserved for interfaces and unions.

Alternatively, if specific names do not work or are unwieldy, nest types instead. For example, the following two schemas are roughly the same in terms of flexibility.

```graphql
type User {
  roleName: String
}
```

```graphql
type User {
  role: UserRole
}

type UserRole {
  name: String
}
```

This extends to scalars as well. An email could be represented as a string, or as an object, like this:

```graphql
type Email {
  address: EmailAddress!
  isVerified: Boolean!
  verifiedAt: ISO8601DateTime
}
```
### Interfaces and Unions

Unions are semantically just marker interfaces (interfaces with zero fields) so I will refer to both interfaces and unions as interfaces.

When choosing field types, prefer interfaces instead of concrete types.

Instead of a `Comment` type, consider a `Comment` interface and a `PostComment` type. Instead of a `Message` type, consider a `Message` interface and a `ChatMessage` type.

### References

Refer to objects, not IDs. GraphQL allows direct traversal so let clients do that.

```graphql
# good
type User {
  invitedBy: User
}

# bad
type User {
  invitedBy: ID
}
```

### Relay

Implement the Relay spec. Read [this](https://relay.dev/docs/en/graphql-server-specification).

One thing often neglected in Relay's cursor pagination is that there is nothing preventing the extension of connections, edges, and page info with additional fields. Cursor pagination can get unwieldy with nested associations. However, this can be avoided by treating your data model as nodes and edges, instead of only using edges for pagination. For example, organization memberships are conceptually edges between users and organizations and should be able to be queried as such.

### Current User

It is very common to provide the current user with more information about themselves and their current session than anyone else can access. This makes it problematic to give the current user direct access to their `User` instance. Instead, create a type specifically for this. Typically, this type is called `Viewer` or `Me`. Add a query field that returns the singleton instance of that type.

```graphql
type Viewer {
  id: ID!
  name: String
}

type Query {
  # ...
  viewer: Viewer
}
```

Alternatively, these fields can live on the query root, but that is messy and, historically, Relay client required that those fields not be root fields. Therefore, `Viewer` or `Me` is standard.


### Arguments

Use field arguments instead of almost identical fields.

It is common to need to provide the client with multiple things that are almost, but not quite, identical. For example, the client might need to access the viewer's avatar in multiple sizes. Instead of adding `avatarSmall`, `avatarMedium`, etc. fields, consider a schema like this:

```graphql
type User {
  avatarURL(input: UserAvatarInput!): URL
}

input UserAvatarInput {
  width: Int!
  height: Int!
}
```

Fields are essentially named-parameter functions. Even though they can take any number of arguments, they should be nullary or unary; they should not take more than one argument. If multiple arguments will be needed, the field should take a single `input` argument that is an input object named `[TypeNameFieldName]Input`. It is best to default to always doing this input object when arguments are needed and only consider scalar input in exceptional circumstances.

### Field Nullability

Circumstances change. What one day is assumed to always be the case may prove to not be.

#### Output

Bias toward nullable output fields. Never make promises to clients that you cannot keep. If you tell them that something will never be null then you close yourself to the possibility that it might, in the future, be null. Prefer nullable fields so that clients are written to handle nulls with good defaults. This is slightly more work upfront for front-end developers but provides long-term flexibility.

#### Input

Bias toward __non__-nullable input fields and arguments. A non-nullable field which was technically optionally can become required without breaking clients. However, a nullable field cannot become non-nullable without breaking clients. Try to have every input field and argument be non-nullable.

### Client-Driven

Move complexity to resolvers (when it does not negatively affect UX).

For example, imagine we have a DB in which we store the user's given name and surname as well as their date of birth and a UI where we show the user's name and age in milliseconds. A good schema might be as follows:

```graphql
type User {
  fullName: String
  dateOfBirth: ISO8601DateTime
}
```
- We provide the client with `fullName`, instead of just `givenName` and `surname`. There is no advantage to the UI concatenating strings, especially when it is complicated by nullable names and potentially multiple languages, some of which have surnames are last names and some of which have surnames are first names. We can provide `givenName` and `surname` fields, but we should also provide the info that clients require instead of making clients jump through hoops to render correctly.
- Even though we are displaying the user's age in milliseconds, we don't expose this field directly. This is because, unlike `fullName`, `ageInMilliseconds` changes constantly. This is a case where the complexity of calculating the age must be done on the client. Calculating the age on the server would cause the UX to suffer.

### Redundant Fields

Do not fear redundant fields.

There is no limit to the number of fields and resolvers your schema has. The only problem posed by additional fields is additional maintenance. This cost is minimal, especially if it allows the clients to do less work. Do not worry about adding fields, especially if it allows you to deprecate other fields.

### Mutations

Mutations are the most difficult part of GraphQL schema design.

#### RPC, not CRUD

This is the key concept. Mutations should be small and generally directly linked to an action the user is taking.

- No: `updateUser(userInput)`, `userUpdate(userInput)`
- Yes: `setUserEmail(emailInput)`, `casUserEmail(currentAndNewEmailInput)`

#### Input

All mutations should take exactly one input argument which should be a non-null input object that is specific to that mutation and named as such.

```graphql
type Mutation {
  registerUser(input: RegisterUserInput!): RegisterUserPayload!
}
```

Mutation input should be specific. The schema should be used to make it impossible to submit obviously incorrect data.

```graphl
input RegisterUserInput {
  email: String!
  password: String!
}
```

But then imagine that it is decided that users should also be able to register with a phone number instead of an email address. The naive approach would be to make `email` optional and add an optional `phone` field:

```graphl
input RegisterUserInput {
  email: String
  phone: String
  password: String!
}
```

This is bad because it allows mutation requests with neither `email` or `phone` and with both `email` and `phone`, neither of which are allowed. What we should do is create a new mutation for this method of registration and, optionally, deprecate the old method.

```graphql
type Mutation {
  registerUser(input: RegisterUserInput!): RegisterUserPayload! @deprecated(reason: "Use `registerUserWithEmail`.")
  registerUserWithEmail(input: RegisterUserWithEmailInput!): RegisterUserWithEmailPayload!
  registerUserWithPhone(input: RegisterUserWithPhoneInput!): RegisterUserWithPhonePayload!
}

input RegisterUserInput {
  email: String! @deprecated(reason: "Use `registerUserWithEmail` mutation.")
  password: String! @deprecated(reason: "Use `registerUserWithEmail` mutation.")
}

input RegisterUserWithEmailInput {
  email: String!
  password: String!
}

input RegisterUserWithPhoneInput {
  email: String!
  password: String!
}
```

This makes it impossible to represent obviously incorrect inputs.

This would be easier if GraphQL had input unions, which might be added in the future.

#### Output

All mutations should return a payload output object that is specific to that mutation and named as such.

```graphql
type Mutation {
  registerUser(input: RegisterUserInput!): RegisterUserPayload!
}
```

All mutation payloads should implement the following interface:

```graphql
interface MutationPayload {
  query: Query!
  # result: MutationResult
}
```

Mutation payloads should always return the root query object so that clients can query from the root, regardless of what the mutation response is otherwise. This can be useful for a variety of reasons, but the primary reason is that full-featured clients like Relay and Apollo need to incorporate the mutation changes into the state but do not know how to by default because the effects of the mutation are not known ahead of time. Clients can use the Query root to query for changed data which should replace stale local data.

Additionally, Mutation payloads should return a `result` union. Please read [this article about result unions](https://medium.com/@sachee/200-ok-error-handling-in-graphql-7ec869aec9bc). Unfortunately, GraphQL's type system cannot express that `MutationPayload` must have a field called `result` and the value must be non-scalar, so we will leave that commented out.

### Files

GraphQL does not specify how to handle files.

#### Download

The easy side of files first -- downloads.

Return signed URLs that expire after a reasonable period.
These URLs should be wrapped in an object that provides additional metadata, like this:

```graphql
type Asset {
  signedUrl: URL!
  signedUrlExpiration: ISO8601DateTime
  mimeType: String
  size: Int
  name: String
}
```

This is a simplified example. It would be better to make `Asset` an interface and then have concrete types like `ImageAsset` and `VideoAsset` that add additional fields like `width`/`height` and `length`.

#### Upload

There seem to be three general strategies for handling file uploads. At the time of writing, I have experience with the first two only.

1. __Base64__ - The easiest way to upload files is by base64-encoding them and passing the file as a string to a mutation. However, this is extremely inefficient for both clients and API servers, does not work for large files, and does not allow multipart/resumable uploads.
2. __Multiple Requests__ - This is where the client uploads the file first, typically to S3 (or another blob store) directly, and then passes a reference to the uploaded file in a mutation. Typically, this is a three-step process of:
   1. Run a mutation like `requestSignedUploadUrl(fileMetadata)` and receive a signed URL.
   2. Upload the file directly to the signed URL.
   3. Run a mutation like `attachImageToPost(postId, uploadedAssetId)`
3. __GraphQL Multipart Request Spec__ - [This](https://github.com/jaydenseric/graphql-multipart-request-spec) is an emerging standard for using GraphQL with `multipart/form-data` content type (instead of the usual `application/json`) and therefore able to transmit files directly. This ships with Apollo Server and some other libraries and tools.

In the spirit of being client-driven, the GraphQL Multipart Request Spec seems like the best approach, currently. It is not a panacea though; it causes new problems like requiring the client to route multipart requests over HTTP even if a WebSocket connection exists and places additional strain on the API server.

## Server

### Schema Creation

SDL (the GraphQL **S**chema **D**efinition **L**anguage) is used throughout this document. It is very useful for describing a schema but too static for easily defining a schema. Do not use SDL or any type of static data (like EDN) for schema definition. Instead, use code to generate the schema (or generate the data which is then used to generate the schema). Good GraphQL schemas tend to be extremely verbose; do not try to define them by hand. See [this article](https://www.prisma.io/blog/the-problems-of-schema-first-graphql-development-x1mn4cb0tyl3) for a more thorough explanation.

### Authorization

Do not rely on parent resolvers for authorization except in scalar resolvers. In other words, object resolvers should not assume that a parent resolver ran first. Doing so makes the schema brittle since it prevents easy addition of fields that return objects for fear that those object resolvers cannot handle multiple schema locations.

Consider the following schema:

```graphql
type Query {
  viewer: Viewer
}

type Viewer {
  projects: [Project!]
}

type Project {
  name: String
  # ...
}
```

In this schema, the `Project` resolvers can safely assume that the viewer is authorized to access the specific project because `Project` resolvers can only run after the `Viewer.projects` resolver runs. However, consider what happens if we add Relay global object identification:

```graphql
interface Node {
  id: ID!
}

extend type Query {
  node(id: ID!): Node
}

extend type Project implements Node {
  id: ID!
}
```
Now, `Query.node` code returns a `Project` and any other field that returns `Node` could return `Project`. The assumption that `Viewer.projects` can authorize access before `Project` resolvers is no longer true.

### Deprecation

Deprecate fields aggressively. Existing clients continue to function. Do not be afraid to deprecate fields, just make sure deprecation reasons point front-end developers to the new field(s), if any, that they should use.

Have a policy about removing deprecated fields. For example, monitor queries in production and remove deprecated fields once they have not been queried in 30 days.

## Client

### Deprecation

Clients should prioritize removing queries that rely on deprecated fields as quickly as possible. The more responsive front-end developers are the faster back-end developers can be to evolve the schema to service the needs of the client.

### Ecosystem

Use a fully-featured stateful GraphQL client like Relay or Apollo. Do not write your own. It is monstrously difficult to do so 100% correctly and performantly.

A huge benefit of GraphQL is the ecosystem. Use it.

### Static Queries

All client-side GraphQL should be static. Do not create dynamic queries by concatenating strings at runtime. All dynamism can be accomplished with variables, directives, and, if necessary, multiple queries.

### Schema Type System

Correctly handle the types that the schema exposes. In GraphQL, a breaking schema change is considered to be any change that causes a query that previously succeeded to fail. If the front-end developers ignore the schema type system it prevents the back-end developers from making changes that should be non-breaking.

There are two common ways in which the schema can, but should now, be ignored or misused.

#### Nullability

If a field is nullable, the client _must_ handle that case, even if values in development are never `null`.

If you use a statically typed language, like TypeScript, generate language types from the GraphQL schema. There are many tools for this. [GraphQL Code Generator](https://graphql-code-generator.com/) may be the most popular.

Complying with the schema is more difficult in dynamically typed languages but best effort should be made. In ClojureScript, [Serene](https://github.com/paren-com/serene) can be used to generate `clojure.spec` specs for the schema and then use those with `test.check` to test UI components.

#### Additions

Adding enum values to existing enum types, adding implementing object types to existing interfaces, and adding object types to existing union types should not be breaking changes. However, they can be breaking changes if the client assumes that these are closed sets. Clients must always assume that the values/members of enums, interfaces, and unions are open sets and have fallback functionality to handle unknown cases.

For example, assume the following schema to start with:

```graphql
enum UserType {
  MECHANIC
  CONTROLLER
}

type Query {
  currentUserType: UserType!
}
```

The client might handle both `UserType` values like this:

```javascript
switch (response.data.currentUserType) {
  case "MECHANIC":
    // do mechanic stuff
    break;
  case "CONTROLLER":
    // do controller stuff
    break;
}
```

Code like this is brittle because it will not correctly handle additive changes to the schema. What if the server adds a `PILOT` `UserType`?

Instead, always code a default case so that the server is always able to make additive changes without breaking existing clients.

```javascript
switch (response.data.currentUserType) {
  case "MECHANIC":
    // do mechanic stuff
    break;
  case "CONTROLLER":
    // do controller stuff
    break;
  default:
    // do default stuff
}
```

This example uses enum values but the same concept applies to interface and union members.

### Local State Management

For non-transient local state, apply local extensions to the schema via the chosen client library. For more information, see [Relay](https://relay.dev/docs/en/local-state-management) or [Apollo](https://www.apollographql.com/docs/react/data/local-state/) documentation.


## Optimization

### Persisted and Whitelisted Queries

Persisted queries are where the client uploads queries to the server once and then sends query IDs instead of query strings at runtime. See [Relay](https://relay.dev/docs/en/persisted-queries) and [Apollo](https://www.apollographql.com/docs/apollo-server/performance/apq/) documentation on their methods of persisted queries.

The Relay method is preferable for private APIs because it allows for queries to be whitelisted, which vastly reduces the surface area for attackers.

When using persisted queries, the `query` field in the JSON request should be replaced by the `id` field which references the ID of the persisted query.