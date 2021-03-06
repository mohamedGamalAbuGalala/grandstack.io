---
id: neo4j-graphql-java
title: Neo4j GraphQL Java
sidebar_label: neo4j-graphql-java
---

# JVM Library to translate GraphQL queries and mutations to Neo4j's Cypher

This is a beta GraphQL transpiler written in Kotlin.

License: Apache 2.

## How does it work

This library

1. parses a GraphQL schema and
2. uses the information of the annotated schema to translate _GraphQL_ queries and parameters into _Cypher_ queries and parameters.

Those Cypher queries can then executed, e.g via the Neo4j-Java-Driver (or other JVM drivers) against the graph database and the results can be returned directly to the caller.

The request, result and error handling is not part of this library, but we provide demo programs on how to use it in different languages.

NOTE: All the [supported features](#features) are listed and explained below, more detailed docs will be added in time.

## FAQ

### How does this relate to the other neo4j graphql libraries?

Similar to `neo4j-graphql-js` this library focuses on query translation, just for the *JVM* instead of Node.js.
It does not provide a server (except as examples) or other facilities but is meant to be used as a dependency included for a single purpose.

If this library is feature complete we plan to replace the code in the current Neo4j server plugin `neo4j-graphql` with a single call to this library.
The server plugin would still handle request-response and error-handling, and perhaps some schema management but be slimmed down to a tiny piece.

### How does this related to graphql-java

This library uses `graphql-java` under the hood for parsing of schema and queries, and managing the GraphQL state and context.
But *not for nested field resolvers or data fetching*.

If you wanted, you could combine `graphql-java` resolvers with this library to have a part of your schema handled by Neo4j.

Thanks a lot to the maintainers of `graphql-java` for the awesome library.

## Usage

You can use the library as dependency: `org.neo4j:neo4j-graphql-java:1.0.0-M03` in any JVM program.

The basic usage should be:

```kotlin
val schema =
        """
        type Person {
            name: ID!
            age: Int
        }
        # Optional if you use generated queries
        type Query {
            person : [Person]
            personByName(name:String) : Person
        }"""

val query = """ { p:personByName(name:"Joe") { age } } """

val schema = SchemaBuilder.buildSchema(idl)
val (cypher, params) = Translator(schema).translate(query, params)

// generated Cypher
cypher == "MATCH (p:Person) WHERE p.name = $pName RETURN p {.age} as p"
```

You find more usage examples in the [TCK scripts](https://github.com/neo4j-graphql/neo4j-graphql-java/tree/master/src/test/resources).

## Demo

Here is a minimalistic example in Groovy using the Neo4j-Java driver and Spark-Java as webserver.
It is running against a Neo4j instance at `bolt://localhost` (username: `neo4j`, password: `password`) containing the `:play movies` graph.

(You can also use a [Kotlin based server example](https://github.com/neo4j-graphql/neo4j-graphql-java/tree/master/src/test/kotlin/GraphQLServer.kt).)

```groovy
// Simplistic GraphQL Server using SparkJava
@Grapes([
  @Grab('com.sparkjava:spark-core:2.7.2'),
  @Grab('org.neo4j.driver:neo4j-java-driver:1.7.2'),
  @Grab('org.neo4j:neo4j-graphql-java:1.0.0-M01'),
  @Grab('com.google.code.gson:gson:2.8.5')
])

import spark.*
import static spark.Spark.*
import com.google.gson.Gson
import org.neo4j.graphql.*
import org.neo4j.driver.v1.*

schema = """
type Person {
  name: ID!
  born: Int
  actedIn: [Movie] @relation(name:"ACTED_IN")
}
type Movie {
  title: ID!
  released: Int
  tagline: String
}
type Query {
    person : [Person]
}
"""

gson = new Gson()
render = (ResponseTransformer)gson.&toJson
def query(value) { gson.fromJson(value,Map.class)["query"] }

graphql = new Translator(SchemaBuilder.buildSchema(schema))
def translate(query) { graphql.translate(query) }

driver = GraphDatabase.driver("bolt://localhost",AuthTokens.basic("neo4j","password"))
def run(cypher) { driver.session().withCloseable { it.run(cypher.query, Values.value(cypher.params)).list{ it.asMap() }}}

post("/graphql","application/json", { req, res ->  run(translate(query(req.body())).first()) }, render);
```
<!-- include::docs/Server.groovy[] -->

Run the example with:

```
groovy docs/Server.groovy
```

and use http://localhost:4567/graphql as your GraphQL URL.

It uses a schema of:

```graphql
type Person {
  name: ID!
  born: Int
  actedIn: [Movie] @relation(name:"ACTED_IN")
}
type Movie {
  title: ID!
  released: Int
  tagline: String
}
type Query {
    person : [Person]
}
```

And can run queries like:

```graphql
{
  person(first:3) {
    name
    born
    actedIn(first:2) {
      title
    }
  }
}
```

image::docs/graphiql.jpg[]

You can also test it with `curl`

```
curl -XPOST http://localhost:4567/graphql -d'{"query":"{person {name}}"}'
```

This example doesn't handle introspection queries but the one in the test directory does.

## Advanced Queries


```
{
  person(filter: {name_starts_with: "L"}, orderBy: "born_asc", first: 5, offset: 2) {
    name
    born
    actedIn(first: 1) {
      title
    }
  }
}
```
_Filter, Sorting, Paging support_

```
{
  person(filter: {name_starts_with: "J", born_gte: 1970}, first:2) {
    name
    born
    actedIn(first:1) {
      title
      released
    }
  }
}
```

## Features

### Current

* parse SDL schema
* resolve query fields via result types
* handle arguments as equality comparisons for top level and nested fields
* handle relationships via @relation directive on schema fields
* @relation directive on types for rich relationships (from, to fields for start & end node)
* handle first, offset arguments
* argument types: string, int, float, array
* request parameter support
* parametrization for cypher query
* aliases
* inline and named fragments
* auto-generate query fields for all objects
* @cypher directive for fields to compute field values, support arguments
* auto-generate mutation fields for all objects to create, update, delete
* @cypher directive for top level queries and mutations, supports arguments

### Next

* skip limit params
* sorting (nested)
* interfaces
* input types
* unions
* scalars
* date(time), spatial

## Documentation

### Parse SDL schema

Currently schemas with object types, enums, fragments and Query types are parsed and handled.
We support @relation directives on fields and types for rich relationships
We support @cypher directives on fields and top-level query and mutation fields.
The configurable augmentation auto-generates queries and mutations (create,update,delete) for all types.
It supports the built-in scalars for GraphQL.
For arguments we support input types in many places and filters as known from GraphCool/Prisma.

### Resolve query Fields via Result Types

For _query fields_ that result in object types (even if wrapped in list/non-null), the appropriate object type is found in the schema and used to translate the query.

e.g.

```graphql
type Query {
  person: [Person]
}
# query "person" is resolved to and via "Person"

type Person {
  name : String
}
```

### Handle Arguments as Equality Comparisons for Top Level and Nested Fields

If you add a simple argument to your top-level query or nested related fields, those will be translated to direct equality comparisons.

```graphql
person(name:"Joe", age:42) {
   name
}
```

to

```cypher
MATCH (person:Person) 
WHERE person.name = 'Joe' AND person.age = 42 
RETURN person { .name } AS person
```

Only that the literal values are turned into parameters.

### Handle Relationships via @relation Directive on Schema Fields

If you want to represent a relationship from the graph in GraphQL you have to add an `@relation` directive that contains the relationship-type and the direction.
Default relationship-type is 'OUT'.
So you can use different domain names in your GraphQL fields that are independent of your graph model.

```graphql
type Person {
  name : String
  actedIn: [Movie] @relation(name:"ACTED_IN", direction:OUT)
}
```

```graphql
person(name:"Keanu Reeves") {
  name
  actedIn {
    title
  }
}
```

NOTE: We use Neo4j's _pattern comprehensions_ to represent nested graph patterns in Cypher.

### Handle first, offset Arguments

To support pagination `first` is translated to `LIMIT` in Cypher and `offset` into `SKIP`
For nested queries these are converted into slices for arrays.

```graphql
person(offset: 5, first:10) {
  name
}
```

```cypher
MATCH (person:Person) 
RETURN person { .name }  AS person 
SKIP 5 LIMIT 10
```

### Argument Types: string, int, float, array

The default Neo4j types are handled both as argument types as well as field types.

NOTE: Datetime and spatial not yet.

### Parameter Support

We handle passed in GraphQL parameters, these are resolved correctly when used within the GraphQL query.

### Parametrization

As we don't want to have literal values in our Cypher queries, all of them are translated into parameters.

```graphql
person(name:"Joe", age:42, first:10) {
   name
}
```

to

```cypher
MATCH (person:Person) 
WHERE person.name = $personName AND person.age = $personAge 
RETURN person { .name } AS person LIMIT $first
```

Those parameters are returned as part of the `Cypher` type that's returned from the `translate()` method.

### Aliases

We support query aliases, they are used as Cypher aliases too, so you get them back as keys in your result records.

For example:

```graphql
query {
  jane: person(name:"Jane") { name, age }
  joe: person(name:"Joe") { name, age }
}
```

### Inline and Named Fragments

This is more of a technical feature, both types of fragments are resolved internally.

### Sorting (top-level)

We support sorting via an `orderBy` argument, which takes an Enum or String value of `fieldName_asc` or `fieldName_desc`.

```graphql
query {
  person(orderBy:[name_asc, age_desc]) {
     name
     age
  }
}
```

```cypher
MATCH (person:Person)
RETURN person { .name, .age } AS person

ORDER BY person.name ASC, person.age DESC
```


NOTE: Those enums are not yet automatically generated. And we don't support ordering yet on nested, related fields.

### @relationship on Types

To represent rich relationship types with properties, a `@relation` directive is supported on an object type.

In our example it would be the `Role` type.

```graphql
type Role @relation(name:"ACTED_IN", from:"actor", to:"movie") {
   actor: Person
   movie: Movie
   roles: [String]
}
type Person {
  name: String
  born: Int
  roles: [Role]
}
type Movie {
  title: String
  released: Int
  characters: [Role]
}
```

```graphql
person(name:"Keanu Reeves") {
   roles {
      roles
      movie {
        title
      }
   }
}
```

### Filters

Filters are a powerful way of selecting a subset of data.
Inspired by the https://www.graph.cool/docs/reference/graphql-api/query-api-nia9nushae[graph.cool/Prisma filter approach], our filters work the same way.

NOTE: we'll create more detailed docs, for now the prisma docs on that topic are pretty good.


We use nested input types for arbitrary filtering on query types and fields

```graphql
{ Company(filter: { AND: { name_contains: "Ne", country_in ["SE"]}}) { name } }
```

You can also apply nested filter on relations, which use suffixes like `("",not,some, none, single, every)`

```graphql
{ Company(filter: {
    employees_none { name_contains: "Jan"},
    employees_some: { gender_in : [female]},
    company_not: null })
    {
      name
    }
}
```

NOTE: Those nested input types are not yet generated, we use leniency in the parser.

### Inline and Named Fragments

We support inline and named fragments according to the GraphQL spec.
Most of this is resolved on the parser/query side.

```graphql
fragment details on Person { name, email, dob }
query {
  person {
    ...details
  }
}
```
_Named Fragment_


```graphql
query {
  person {
    ... on Person { name, email, dob }
  }
}
```
_Inline Fragment_


### @cypher Directives

With `@cypher` directives you can add the power of Cypher to your GraphQL API.
It allows you, without code to compute field values using complex queries.
You can also write your own, custom top-level queries and mutations using Cypher.

Arguments on the field are passed to the Cypher statement as parameters.
Input types are supported, they appear as `Map` type in your Cypher statement.

NOTE: Those Cypher directive queries are only included in the generated Cypher statement if the field or query is included in the GraphQL query.

#### On Fields

@cypher directive on a field

```graphql
type Movie {
  title: String
  released: Int
  similar(limit:Int=10): [Movie] @cypher(statement:"
        MATCH (this)-->(:Genre)<--(sim)
        WITH sim, count(*) as c ORDER BY c DESC LIMIT $limit
        RETURN sim")
}
```

Here the `this` variable is bound to the current movie and you can use it to navigate the graph and collect data.
The `limit` variable is passed to the query as parameter.

#### On Queries

Similarly you can use the `@cypher` directive with a top level query.


```graphql
type Query {
   person(name:String) Person @cypher("MATCH (p:Person) WHERE p.name = $name RETURN p")
}
```
_@cypher directive on query_

Of course you can also return arrays from your query, the statements on query fields should be read-only queries.

#### On Mutations

You can do the same for mutations, just with updating Cypher statements.


```graphql
type Mutation {
   createPerson(name:String, age:Int) Person @cypher("CREATE (p:Person) SET p.name = $name, p.age = $age RETURN p")
}
```

_@cypher directive on mutation_

You can use more complex statements for creating these entities or even subgraphs.

NOTE: The common CRUD mutations and queries are auto-generated, see below.

### Auto Generate Queries and Mutations

To reduce the amount of boilerplate code a user has to write we auto-generate top-level CRUD queries and mutations for all types.

This is configurable via the API, you can:

* disable auto-generation (for mutations/queries)
* disable it per type
* disable mutations per operation (create,delete,update)

For a schema like this:

```graphql
type Person {
   id:ID!
   name: String
   age: Int
   movies: [Movie]
}
```


It would auto-generate quite a lot of things:

* a query: `person(id:ID, name:String , age: Int, _id: Int, filter:_PersonFilter, orderBy:_PersonOrdering, first:Int, offset:Int) : [Person]`
* a `_PersonOrdering` enum, for the `orderBy` argument with all fields for `_asc` and `_desc` sort order
* a `_PersonInput` for creating Person objects
* a `_PersonFilter` for the `filter` argument, which is a deeply nested input object (see <<filters>>)
* mutations for:
** createPerson: `createPerson(id:ID!, name:String, age: Int) : Person`
** mergePerson:  `mergePerson(id:ID!,  name:String, age:Int) : Person`
** updatePerson: `updatePerson(id:ID!, name:String, age:Int) : Person`
** deletePerson: `deletePerson(id:ID!) : Person`
** addPersonMovies: `addPersonMovies(id:ID!,movies:[ID!]!) : Person`
** deletePersonMovies: `deletePersonMovies(id:ID!,movies:[ID!]!) : Person`

You can then use those in your GraphQL queries like this:

```graphql
query { person(age:42, orderBy:name_asc) {
   id
   name
   age
}
```

or


```graphql
mutation {
  createPerson(id: "34920n9qw0", name:"Jane Doe", age:42) {
    id
    name
    age
  }
}
```
