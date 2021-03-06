# Slowly-Changing Dimension Store (SCDS)

[![Build Status](https://travis-ci.org/chop-dbhi/scds.svg?branch=master)](https://travis-ci.org/chop-dbhi/scds) [![Coverage Status](https://coveralls.io/repos/chop-dbhi/scds/badge.svg?branch=master&service=github)](https://coveralls.io/github/chop-dbhi/scds?branch=master) [![GoDoc](https://godoc.org/github.com/chop-dbhi/scds?status.svg)](https://godoc.org/github.com/chop-dbhi/scds)

SCDS is designed to answer a single question, "has this data changed since the last time I saw it?" The motivation stems from working on data integration pipelines where it may be unknown when or how downstream data has changed. There are two basic use cases:

- Check the data to fail quickly or apply other logic.
- Put the data each time it is seen to see how it changes over time.

*Note: albeit functional, this is a prototype implementation for solving this problem.*

## Usage

There are two interfaces supported, command line and HTTP. They share the same set of operations and work with JSON-encoded data.

### Operations

#### `put`

Put an object in the store where `value` is a valid JSON document. Putting the same object consecutively will not result in duplicate changes.

```
put <key> <value>
```

#### `get`

Get the current state of the object. Use the `-version` or `-time` option to get a particular revision.

```
get <key>
```

#### `keys`

Gets a list of keys in the store.

```
keys
```

#### `log`

Get the log of changes for an object.

```
log <key>
```

#### `config`

Prints the configuration options used.

```
config
```

#### `subscribe`

Subscribes one or more email addresses for notifications.

```
subscribe email [emails...]
```

#### `unsubscribe`

Unsubscribes one or more email addresses from notifications.

```
unsubscribe email [emails...]
```

### CLI

See `scds help` for more information.

Inline JSON.

```bash
scds put bob '{"name": "Bob"}'
```

```json
{
  "version": 1,
  "time": 1436960622,
  "additions": {
    "name": "Bob"
  },
  "removals": null,
  "changes": null
}
```

Alternately, if not `value` is supplied, data will be read from stdin.

```bash
scds put hello < hello.json
```

Running the above command again will return nothing since nothing changed. However if we change it a new revision will be created.

```bash
scds put bob '{"name": "Bob Smith", "email": "bob@smith.net"}'
```

```json
{
  "version": 2,
  "time": 1436960632,
  "additions": {
    "email": "bob@smith.net"
  },
  "removals": null,
  "changes": {
    "name": {
      "before": "Bob",
      "after": "Bob Smith"
  }
}
```

To get the current state of the object use `get`.

```bash
scds get bob
```

```json
{
  "key": "bob",
  "value": {
    "email": "bob@smith.net",
    "name": "Bob Smith"
  },
  "version": 2,
  "time": 1436960632
}
```

To get the log of changes over time:

```
scds log bob
```

```json
[
  {
    "version": 1,
    "time": 1436960622,
    "additions": {
      "name": "Bob"
    },
    "removals": null,
    "changes": null
  },
  {
    "version": 2,
    "time": 1436960632,
    "additions": {
      "email": "bob@smith.net"
    },
    "removals": null,
    "changes": {
      "name": {
        "before": "Bob",
        "after": "Bob Smith"
      }
    }
  }
]
```

### HTTP

Start the HTTP server.

```bash
scds http
* [http] Listening on locahost:5000
```

The input and output of the endpoints match the command-line interface.

- `GET /keys`
- `PUT /objects/<key>`
- `GET /objects/<key>`
- `GET /objects/<key>/v/<version>`
- `GET /objects/<key>/t/<time>`
- `GET /log/<key>`


## Notifications

SCDS has built-in support for basic email notifications when new objects are added or when objects change. To subscribe, use the `subscribe` command.

```
scds subscribe <email>
```


## Dependencies

- MongoDB


## Configuration

Configuration options can be supplied in a file, as environment variables, or command-line arguments (following that precedence). The default configuration options are listed below (in a YAML format).

```yaml
debug: false
config: ""
mongo:
  uri: localhost/scds
http:
  host: localhost
  port: 5000
  tlscert: ""
  tlskey: ""
  cors: false
smtp:
  host: localhost
  port: 25
  user: ""
  password: ""
  from: ""
```

Environment variables are prefixed with `SCDS_`, are uppercased, and nested options are delimited with an underscore. For example, `SCDS_MONGO_URI` would set the `uri` option in the `mongo` map. Alternately, the command-line flag can be supplied:

```
scds -mongo.uri dockerhost/scds ...
```

If a `scds.yml` file is defined in the working directory, it will be read in automatically. To use an alternate path, the `-config <path>` (or `SCDS_CONFIG=<path>`) can be used.

### JSON Schema

SCDS supports document validation against predefined [JSON Schema](http://json-schema.org) documents. The simplest setup is a schema used for all documents.

```yaml
schemas:
  default:
    file: schema.json
```

To validate a subset of objects, a `scope` can be specified at the `object` or `value` level. An `object` scope supports the `pattern` key for matching on the `key` of the object.

```yaml
schemas:
  users:
    scope: object
    pattern: "users\..*"
    file: user.json
```

Any object with a key that starts with `users.` will be validated by the `user.json` schema. Thus the following would match:

```http
PUT /objects/users.1

{ ... }
```

or via the command line:

```
scds put users.1 '{}'
```

The `value` scope pattern matches against a particular field in the object data.

```yaml
schemas:
  users:
    scope: value
    field: type
    pattern: user
    file: user.json
```

The following document would match since it contains the `type` field with a value of `user`.

```http
PUT /objects/abc123

{
  "type": "user",
  "firstName": "Jon",
  "lastName": "Doe"
}
```

An object that matches multiple schema patterns will be validated against all of them. This allows for composing smaller schemas together to validate various parts of the documents.

## Docker

The image defaults to running the HTTP interface and looks for a MongoDB server listening on `mongo:27017`.

```
docker run -it --link mongo:mongo -p 5000:5000 dbhi/scds
```

### Compose

A basic Docker Compose file is provided that includes starting a MongoDB container, however it should be changed to mount a volume on the host so the data is persisted.

```
docker-compose up -d
```

## Development

Go 1.7 or later is required for development.
