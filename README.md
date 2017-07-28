# Apollo Tracing

Apollo Tracing is a proposed GraphQL response format extension to expose trace data for requests.

To support tracing, GraphQL servers should include resolver timings and other execution information as part of the GraphQL response, and tools can extract this data from the response to give more insight into individual requests, and to track server performance and schema usage over time.

The format is work in progress, and we're collaborating with others in the GraphQL community to make it broadly available, and to build awesome tools on top of it.

### Implementations

- [Node.js](https://github.com/apollographql/apollo-tracing-js)
- [Sangria (Scala)](https://gist.github.com/OlegIlyenko/124b55e58609ad45fcec276f15158d16)

### Use in Apollo Optics

One use of Apollo Tracing is to add support for [Apollo Optics](https://www.apollodata.com/optics/) to more GraphQL servers.

Currently, Apollo Optics relies on an agent running in a server that collects, aggregates, and batches up data to send to the Optics backend. Because agents contain fairly complicated logic, we've only been able to support Node.js and Ruby servers so far.

In the new architecture, trace data is included with the GraphQL response, and a separate proxy process (provided by Apollo) is responsible for filtering out the trace data and performing the aggregation and batching. This will make it much easier to use Optics with every GraphQL server, as long as it supports Apollo Tracing.

## Resonse Format

The GraphQL specification allows servers to [include additional information as part of the response under an `extensions` key](https://facebook.github.io/graphql/#sec-Response-Format):
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

Some more details (adapted from [the description of the JSON encoding of Protobuf's Timestamp type](https://github.com/google/protobuf/blob/master/src/google/protobuf/timestamp.proto)):

> A timestamp is encoded as a string in the [RFC 3339](https://www.ietf.org/rfc/rfc3339.txt) format. That is, the format is "{year}-{month}-{day}T{hour}:{min}:{sec}[.{frac_sec}]Z" where {year} is always expressed using four digits while {month}, {day}, {hour}, {min}, and {sec} are zero-padded to two digits each. The fractional seconds, which can go up to 9 digits (i.e. up to 1 nanosecond resolution), are optional. The "Z" suffix indicates the timezone ("UTC"); the timezone is required, though only UTC (as indicated by "Z") is presently supported.
For example, "2017-01-15T01:30:15.01Z" encodes 15.01 seconds past 01:30 UTC on January 15, 2017.
In JavaScript, one can convert a Date object to this format using the standard [`toISOString()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString) method. In Python, a standard `datetime.datetime` object can be converted to this format using [`strftime`](https://docs.python.org/2/library/time.html#time.strftime) with the time format spec '%Y-%m-%dT%H:%M:%S.%fZ'. Likewise, in Java, one can use the Joda Time's [`ISODateTimeFormat.dateTime()`](http://joda-time.sourceforge.net/apidocs/org/joda/time/format/ISODateTimeFormat.html#dateTime()) to obtain a formatter capable of generating timestamps in this format.

- Resolver timings should be collected in nanoseconds using a monotonic clock like `process.hrtime()` in Node.js or `System.nanoTime()` in Java.
> The limited precision of numbers in JavaScript is not an issue for our purposes, because `Number.MAX_SAFE_INTEGER` nanoseconds is about 104 days, which should be plenty even for long running requests!

- The server should keep the start time of the request both as wall time, and as monotonic time to calculate `startOffset`s and `duration`s (for the request as a whole and for individual resolver calls, see below).

- The `duration` of a request is relative to the request start.

- The `startOffset` of a resolver call is relative to the request start.

- The `duration` of a resolver call is relative to the resolver call start.

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

Although we tried other approaches to make the tracing format more compact (including deduplication of keys, commmon items, and structure) this complicated generating and interpreting trace data, and didn't bring the size down as much as compressing the entire HTTP response body does.

In our tests on Node.js, the processing overhead of compression is less than the overhead of sending additional bytes for an uncompressed response. But more test results from different server environments are definitely welcome, so we can help people make an informed decision about this.