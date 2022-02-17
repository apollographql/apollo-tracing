# Apollo Tracing

---

> **[2022-02-16] Notice:** This tracing format was designed to provide tracing data from graphs to the Apollo Engine `engineproxy`, a project which was retired in 2018. We learned that a trace format which describes resolvers with a flat list of paths (with no way to aggregate similar nodes or repeated path prefixes) was inefficient enough to have real impacts on server performance, and so we have not been actively developing consumers or producers of this format for several years. Apollo Server (as of v3) no longer ships with support for producing this format, and `engineproxy` which consumed it is no longer supported. We suggest that people looking for formats for describing performance traces consider either the [Apollo Studio protobuf-based trace format](https://www.apollographql.com/docs/studio/metrics/usage-reporting/#tracing-format) or a more generic format such as OpenTelemetry.
 
---

Apollo Tracing is a GraphQL extension for performance tracing.

Thanks to the community, Apollo Tracing already works with most popular GraphQL server libraries, including Node, Ruby, Scala, Java, Elixir, Go and .NET, and it enables you to easily get resolver-level performance information as part of a GraphQL response.

Apollo Tracing works by including data in the extensions field of the GraphQL response, which [is reserved by the GraphQL spec for extra information that a server wants to return](https://spec.graphql.org/June2018/#sec-Response-Format). That way, you have access to performance traces alongside the data returned by your query.

It’s already supported by [Apollo Engine](https://www.apollographql.com/engine/), and we’re excited to see what other kinds of integrations people can build on top of this format.

We think this format is broadly useful, and we’d love to work with you to add support for it to your tools of choice. If you’re looking for a first idea, we especially think it would be great to see support for Apollo Tracing in [GraphiQL](https://github.com/graphql/graphiql) and the [Apollo Client developer tools](https://github.com/apollographql/apollo-client-devtools)!

If you’re interested in working on support for other GraphQL servers, or integrations with more tools, please get in touch on the `#apollo-tracing` channel on the [Apollo Slack](http://www.apollodata.com/#slack).

## Supported GraphQL Servers

- [Node.js](https://github.com/apollographql/apollo-server/tree/HEAD/packages/apollo-tracing)
- Python
  - [ariadne](https://ariadnegraphql.org/docs/apollo-tracing)
  - [Strawberry](https://strawberry.rocks/docs/operations/tracing#apollo)
- [Ruby](https://github.com/uniiverse/apollo-tracing-ruby)
- Scala
  - [Sangria](https://github.com/sangria-graphql/sangria-slowlog#apollo-tracing-extension)
  - [Caliban](https://ghostdogpr.github.io/caliban/docs/middleware.html#wrapper-types)
- [Java](https://github.com/graphql-java/graphql-java/pull/577)
- [Elixir](https://github.com/sikanhe/apollo-tracing-elixir)
- .NET
  - [graphql-dotnet](https://graphql-dotnet.github.io/docs/getting-started/metrics)
  - [Hot Chocolate](https://hotchocolate.io/docs/apollo-tracing)
- [PHP/Laravel](https://lighthouse-php.com/master/performance/tracing.html)
- [Go](https://github.com/99designs/gqlgen/tree/master/graphql/handler/apollotracing)

## Response Format

The GraphQL specification allows servers to [include additional information as part of the response under an `extensions` key](https://spec.graphql.org/June2018/#sec-Response-Format):
> The response map may also contain an entry with key `extensions`. This entry, if set, must have a map as its value. This entry is reserved for implementors to extend the protocol however they see fit, and hence there are no additional restrictions on its contents.

Apollo Tracing exposes trace data for an individual request under a `tracing` key in `extensions`:

```
{
  "data": <>,
  "errors": <>,
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": <>,
      "endTime": <>,
      "duration": <>,
      "parsing": {
        "startOffset": <>,
        "duration": <>,
      },
      "validation": {
        "startOffset": <>,
        "duration": <>,
      },
      "execution": {
        "resolvers": [
          {
            "path": [<>, ...],
            "parentType": <>,
            "fieldName": <>,
            "returnType": <>,
            "startOffset": <>,
            "duration": <>,
          },
          ...
        ]
      }
    }
  }
}
```

### Collected data

- The `startTime` and `endTime` of the request are timestamps in [RFC 3339](https://www.ietf.org/rfc/rfc3339.txt) format with at least millisecond but up to nanosecond precision (depending on platform support).

Some more details (adapted from [the description of the JSON encoding of Protobuf's Timestamp type](https://github.com/google/protobuf/blob/HEAD/src/google/protobuf/timestamp.proto)):

> A timestamp is encoded as a string in the [RFC 3339](https://www.ietf.org/rfc/rfc3339.txt) format. That is, the format is "{year}-{month}-{day}T{hour}:{min}:{sec}[.{frac_sec}]Z" where {year} is always expressed using four digits while {month}, {day}, {hour}, {min}, and {sec} are zero-padded to two digits each. The fractional seconds, which can go up to 9 digits (i.e. up to 1 nanosecond resolution), are optional. The "Z" suffix indicates the timezone ("UTC"); the timezone is required, though only UTC (as indicated by "Z") is presently supported.
For example, "2017-01-15T01:30:15.01Z" encodes 15.01 seconds past 01:30 UTC on January 15, 2017.
In JavaScript, one can convert a Date object to this format using the standard [`toISOString()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString) method. In Python, a standard `datetime.datetime` object can be converted to this format using [`strftime`](https://docs.python.org/2/library/time.html#time.strftime) with the time format spec '%Y-%m-%dT%H:%M:%S.%fZ'. Likewise, in Java, one can use the Joda Time's [`ISODateTimeFormat.dateTime()`](http://joda-time.sourceforge.net/apidocs/org/joda/time/format/ISODateTimeFormat.html#dateTime()) to obtain a formatter capable of generating timestamps in this format.

- Resolver timings should be collected in nanoseconds using a monotonic clock like `process.hrtime()` in Node.js or `System.nanoTime()` in Java.
> The limited precision of numbers in JavaScript is not an issue for our purposes, because `Number.MAX_SAFE_INTEGER` nanoseconds is about 104 days, which should be plenty even for long running requests!

- The server should keep the start time of the request both as wall time, and as monotonic time to calculate `startOffset`s and `duration`s (for the request as a whole and for individual resolver calls, see below).

- The `duration` of a request is in nanoseconds, relative to the *request start*, as an integer.

- The `startOffset` of parsing, validation, or a resolver call is in nanoseconds, relative to the *request start*, as an integer.

- The `duration` of parsing, validation, or a resolver call is in nanoseconds, relative to the *resolver call start*, as an integer.
> The end of a resolver call represents the return of a value for a field, but it does not include resolving subfields. If an asynchronous value such as a promise is returned from a resolver however, the resolver call isn't considered to have ended until the asynchronous value has been resolved.

- The `path` is the response path of the current resolver in a format similar to the error result format specified in the GraphQL specification:
> This field should be a list of path segments starting at the root of the response and ending with the field associated with the error. Path segments that represent fields should be strings, and path segments that represent list indices should be 0‐indexed integers. If the error happens in an aliased field, the path to the error should use the aliased name, since it represents a path in the response, not in the query.

- `parentType`, `fieldName` and `returnType` are strings that reflect the runtime type information usually passed to resolvers (e.g. in the `info` argument for `graphql-js`).

### Example

```graphql
query {
  hero {
    name
    friends {
      name
    }
  }
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        },
        {
          "name": "Han Solo"
        },
        {
          "name": "Leia Organa"
        }
      ]
    }
  },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2017-07-28T14:20:32.106Z",
      "endTime": "2017-07-28T14:20:32.109Z",
      "duration": 2694443,
      "parsing": {
        "startOffset": 34953,
        "duration": 351736,
      },
      "validation": {
        "startOffset": 412349,
        "duration": 670107,
      },
      "execution": {
        "resolvers": [
          {
            "path": [
              "hero"
            ],
            "parentType": "Query",
            "fieldName": "hero",
            "returnType": "Character",
            "startOffset": 1172456,
            "duration": 215657
          },
          {
            "path": [
              "hero",
              "name"
            ],
            "parentType": "Droid",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 1903307,
            "duration": 73098
          },
          {
            "path": [
              "hero",
              "friends"
            ],
            "parentType": "Droid",
            "fieldName": "friends",
            "returnType": "[Character]",
            "startOffset": 1992644,
            "duration": 522178
          },
          {
            "path": [
              "hero",
              "friends",
              0,
              "name"
            ],
            "parentType": "Human",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 2445097,
            "duration": 18902
          },
          {
            "path": [
              "hero",
              "friends",
              1,
              "name"
            ],
            "parentType": "Human",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 2488750,
            "duration": 2141
          },
          {
            "path": [
              "hero",
              "friends",
              2,
              "name"
            ],
            "parentType": "Human",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 2501461,
            "duration": 1657
          }
        ]
      }
    }
  }
}
```

### Compression

We recommend that people enable compression in their GraphQL server, because the tracing format adds to the response size, but compresses well.

Although we tried other approaches to make the tracing format more compact (including deduplication of keys, common items, and structure) this complicated generating and interpreting trace data, and didn't bring the size down as much as compressing the entire HTTP response body does.

In our tests on Node.js, the processing overhead of compression is less than the overhead of sending additional bytes for an uncompressed response. But more test results from different server environments are definitely welcome, so we can help people make an informed decision about this.
