# mimi-messaging-spec

TLDR - it's a proposal for a YAML based API specification format,
which is machine readable and flexible enough, and concise and simple to be
human readable enough.

## Messaging API specification

The applications utilizing Messaging layer for inter-service communications
collectively implement some kind of API -- they respond to some COMMANDs and QUERYs
in an expected format, they produce EVENTs that have some predefined structure.

When it comes to API specifications and tools for documenting message based APIs,
the choice is limited (basically there's only one: AsyncAPI).
AsyncAPI's goal is to document asynchronous communication patterns
(e.g. EVENTs), and it is not designed for the synchronous case.

Here we propose a new API specification format, that can be used to document the
Messaging API, covering COMMAND, QUERY and EVENT communication patterns.


## Conventions

### COMMAND, QUERY and EVENT

The API specification is designed to document COMMAND, QUERY and EVENT
communication patterns.

See: [Messaging layer properties](https://github.com/kukushkin/mimi-messaging/blob/master/docs/Messaging_Layer_Properties.md)

### Request

*request* -- is a COMMAND or QUERY.

It is assumed that a QUERY request target can be invoked also as a COMMAND.
In that case the response is simply discarded. The accepted parameters
schema for a request target is assumed to be the same irrespectively of
how the target is invoked -- as a COMMAND or a QUERY.

### Target

*target* -- is a string that describes the destination of the COMMAND or QUERY,
or the topic and event type of the EVENT.

COMMAND and QUERY targets have the following format:

```
<queue_name>/<method_name>
```

Where `queue_name` is a String of characters from the following alphabet:
`[A-Za-z0-9\_\-\.]`.
`method_name` is a String, which is a valid identifier: `[A-Za-z\_][A-Za-z0-9]*`

For example:
```
customers/create
bookkeeping.transactions/list
```

EVENT targets have the following format:
```
<topic_name>#<event_type>
```

Where `topic_name` is a String of characters from the following alphabet:
`[A-Za-z0-9\_\-\.]`.
`event_type` is a String, which is a valid identifier: `[A-Za-z\_][A-Za-z0-9]*`

For example:
```
customers#created
bookkeeping.transactions#updated
```

### Message payload

Any kind of message used in communication patterns is assumed to be an Object
(understand as JSON Object, Ruby Hash): params to a request,
a request response, and an event.

## API specification format

The goal of the proposed API specification format is to allow to describe
a Messaging API implemented by the applications in a machine readable format first,
human readable -- second.

The intent is to have an API specification as a concise document with high
signal-to-noise ratio, which is achieved by avoiding the excessive use of keywords.
(In contrast to JSON schema format, for example).

### It's a YAML

An API specification is a collection of YAML documents, each one of which contains
an Object (understand as JSON Object, Ruby Hash) as a top level value.

A single YAML document describes Messaging API targets as keys of this top level Object.

For request targets, input parameters and response schema (for QUERY requests) is documented.

For event targets, the event schema is documented.


Example:
```yaml
#
# Comments in the YAML document are intended to improve human-readability
# of the specification
#

# This target describes a COMMAND/QUERY request
customers/create:
  params:                   # describes required/accepted parameters of the request
    id: :string
    first_name: :string
    last_name: :string
  return:                   # describes a response schema for QUERY requests
    id: :string
    first_name: :string
    last_name: :string
    created_at: :timestamp
    updated_at: :timestamp

# This target describes an EVENT
customers#created:
  id: :string
  first_name: :string
  last_name: :string
  created_at: :timestamp
  updated_at: :timestamp

```

### Documenting COMMAND and QUERY

To document a COMMAND or QUERY request we start with specifying the request target
as a top level key. The simplest specification of a COMMAND/QUERY request
looks like a request target as a top level key and a null value:

```yaml
customers/create:
```

This indicates that Messaging API implements this request target,
and it accepts any message as input parameters and can produce any message
as a response, including no response in case of COMMAND-only request.

#### Documenting input parameters and response

To define schemas for input parameters and the response, we specify
the request target and an Object as its value, having the keys `params` (mandatory)
and `return` (optional). In the simplest case, their respective values
are left as null:

```yaml
customers/create:
  params:
  return:

customers/lock:
  params:
```

`params` key with a null value indicates that any parameters are accepted.

`return` key with a null value indicates that the response WILL be produced,
and it can be any message.

Note, that the `return` key is optional, and not specifying it means that
this request can only be invoked as COMMAND and will not produce any response.
In the example above `customers/create` can be invoked as both COMMAND and QUERY,
while `customers/lock` can only be invoked as COMMAND.

To specify some actual input parameters or response schema, we have to provide
some non-null value to `params` and `return` keys. As a simple example here
we are going to describe schemas as Objects with some keys and values:

```yaml
customer/create:
  params:           # As request parameters, a message with a single
    name: :string   # attribute "name" and a string value is accepted

  return:           # As response, a message containing an Object
    id: :string     # with following keys and values will be returned
    name: :string
    created_at: :timestamp
    updated_at: :timestamp
```

A complete description on how to specify message schemas can be found
later in this document.

### Documenting EVENT

Documenting EVENTs works in a similar way -- we specify the event target
as a top level key. A simplest EVENT specification is an event target
as a top level key which has a null value:

```yaml
customers#created:
```

This indicates that this EVENT may be produced, and its message
can be anything.

To specify the actual message schema, we  provide some non-null value
to an event target key. For example:

```yaml
customers#created:
  id: :string
  name: :string
  created_at: :timestamp
  updated_at: :timestamp
```

### Defining Message Schemas

A message schema can be defined in one of the following ways:

* as an Object
* as a type reference
* as a union of message schema definitions

An Object defines the actual structure of the message attributes and values.

A *type reference* is a string literal starting with ":" followed
by some type name, and it indicates that the schema is defined as
a custom type somewhere else in API specification.

A union of message schema definitions is represented as a YAML array
of other message schema definitions.

For example:

```yaml
# message schema is defined as an Object
customers#created:
  id: :string
  name: :string
  created_at: :timestamp
  updated_at: :timestamp

# message schema is defined as a type reference
customers#updated: :customer

# message schema is defined as a union of message schema definitions,
# in this case: type references
transactions#updated:
  - :transaction_deposit
  - :transaction_withdrawal
  - :transaction_transfer
```
#### Object

When a message schema is defined as an Object, it specifies which
attributes MUST be present in the message and the type of their
value.

For example:
```yaml
customers#created:
  id: :string
  name: :string
  created_at: :timestamp
  updated_at: :timestamp
```

Attributes listed in the message schema definition MUST be present,
and their value types MUST follow the specification.

Attributes not listed in the message schema definition MAY be present,
and their value types are unknown.

When specifying a message schema as an Object, each attribute is defined as:
```
<attribute_name>: <type>
```

The `<type>` can be specified in multiple ways, described below:

##### type: literal

```
<attribute_name>: <literal>
```

A simplest case is when the value is specified as a single literal: string, integer
or boolean. It means that the attribute can only take this exact value.

For example:

```yaml
enabled: true
```

Here the `enabled` attribute of the message is always a boolean value of `true`.


##### type: type reference

```
<attribute_name>: <type_reference>
```

A *type reference*, as in the message schema, is a string literal which starts
with ":" followed by some type name. It refers some built-in or a custom data type.

For example:
```yaml
created_at: :timestamp
```

It describes a `created_at` attribute that can take any value of type :timestamp.
See [built-in types](#built-in-data-types).

##### type: object


```
<attribute_name>:
  <attribute1_name>: <type>
  <attributeN_name>: <type>
```

When the type of an attribute is specified as an Object, it indicates this attribute
value is a nested Object with the given scheme.

Example:
```yaml
customers#updated:
  id: :string
  address:
    country: :string
    city: :string
```

##### type: :array

```
<attribute_name>:
  :array: <type>
```

There is an exception to the previous case: when the attribute type is
specified as an Object, but this Object has a single attribute in the form of
`:array: <type>`, then it indicates a value which is an array of elements
of given type. The `<type>` can also be null, which then indicates an array
of elements of any type.

Example 1:
```yaml
signatures:
  :array: :string
```

This defines a `signatures` attribute, which value is an array of strings.

Example 2:
```yaml
errors:
  :array:
    code: :string
    message: :string
```

This defines an `errors` attribute, which is an array of objects with `code` and `message` attributes.

##### type: union

A union of types is represented as a YAML array, where each element is a `<type>`:
```
<attribute_name>:
  - <type>
  - <type>
```

For example:

```yaml
value:       # defines a "value" attribute, which can be a string, an integer or a null
  - :string
  - :integer
  - :null
```

Of course, any kind of type definition can be specified in a union, and they can be mixed:

```yaml
state:       # defines a "state" attribute, which can take values "PENDING" or "COMPLETED"
 - "PENDING" # or can be any other (undocumented) string value
 - "COMPLETED"
 - :string
```

##### type: extended base type

TBD: how the base types can be extended?

For example:
```
<attribute_name>:
  :string:
    pattern: "^[A-Za-z0-9\_]+$"
```

### Built-in Data Types

There is a number of built-in data types defined by the specification, listed
here as type references:

* `:null` -- represents a null value
* `:string` -- any UTF-8 string value, including an empty string
* `:integer` -- any integer value
* `:timestamp` -- a timestamp as a string in ISO8601 format
* `:boolean` -- a `true` or `false` literal
* `:object` -- represents an Object with undefined attributes
* `:array` -- represents an Array with undefined elements

TBD:
* `:decimal` -- a decimal value as a string in a decimal (i.e. NOT scientific) notation, e.g. "123.456789"


### Custom Data Types

Whenever the same message schema or some value type specification is reused
accross many different request and event targets, it is convenient to
define this as a new named data type, and use this name in type references.

Custom data types are specified as top level keys in API specification, which have
the following format:

```
:<type_name>: <type>
```


### Extensions: optional values

In the way we have discussed so far, optional values can be represented
as union types including `:null` and some other type:

```yaml
name:
  - :null
  - :string
```

On the other hand an optional value is a common case and using a union type
just for this seems like overkill. As a more concise way to indicate a nullable
value, we introduce a following convention:

> A type reference followed by `?` indicates the value can either be of referenced type
> or null.

For example, in this case `name` can either be a string or null:
```yaml
name: :string?
```

### Extensions: optional keys

In a similar and as common case as optional values, any message can contain optional
keys, which currently can be represented as a union of Objects with different set of attributes:

```yaml
params:
  - id: :string            # in this example of input parameters schema definition
    first_name: :string    # an "id" attribute is optional.
    last_name: :string
  - first_name: :string
    last_name: :string
```

Again, especially in the case where we have many optional attributes, we have
to define all the possible combinations in a union, which is obviously an overkill.

To avoid this and to drive us closer to a more concise API specification,
we introduce another convention:

> An Object attribute name followed by `?` indicates that this attribute
> MAY or MAY NOT be present in the message

Then, the `params` example above can be rewritten in a much shorter way:

```yaml
params:
  id?: :string
  first_name: :string
  last_name: :string
```

To illustrate this approach, in the following artificial example we define
an attribute which may not be present, and if it is, it may contain a null
value or a string:

```yaml
description?: :string?
```

## API specification grammar

An attempt to define a format grammar in pseudo-code:

```yaml

# COMMAND and QUERY requests specification
<request_spec> = <request_target>: <request_def> | null
<request_def> = # YAML-like arrays and expressions with "|" indicate a union
  - params: <message_spec> | null
  - params: <message_spec> | null
    return: <message_spec> | null

# EVENT specification
<event_spec> = <event_target>: <message_spec> | null

<message_spec> =
  - <object_spec>
  - <type_reference>      # TBD: Only references to Object types should be allowed here
  - <array of <message_spec> >

<object_spec> =           # Example:
  <attr1>: <type>         #   id:   :integer
  <attrN>: <type>         #   name: :string

<type> =
  - <literal>
  - <type_reference>
  - <object_spec>
  - <array_spec>
  - <extended_type_spec> # TBD
  - <array of <type> >

<literal> =
  - <string_literal>
  - <integer_literal>
  - <boolean_literal>

<type_reference> = :<type_name> # note the ":" characted in front, e.g. ":string"

<array_spec> =
  :array: <type>


# Custom type specification, defines a new type with a name <type_name>
<type_spec> = :<type_name>: <type> | null

# TBD: Extending base types, like :string and :integer
<extendend_type_spec> =
  - <extended_string_type_spec>
  - <extended_integer_type_spec>
  # more to follow?

<extended_string_type_spec> =
  :string:
    pattern: <regexp> # TBD
    size:  <integer> # TBD
    max_size: <integer> # TBD

```

## Full Example

```yaml
#
# This document describes "customers" resource API
#

# defines a new type :uid, which is a string of 32 hexadecimal characters
:uid:
  :string:
    pattern: "^[0-9a-f]{32}$"

# defines a new type :customer, which is a Customer representation
:customer:
  id: :uid
  first_name: :string
  last_name: :string
  created_at: :timestamp
  updated_at: :timestamp


# Creates a new Customer
#
# Broadcasts: customers#created, customers#updated
#
customers/create:
  params:
    id?: :uid    # optional attribute, a client-defined ID
    first_name: :string
    last_name: :string
  return: :customer

# Updates an existing Customer
#
# Broadcasts: customers#updated
#
customers/update:
  params:
    id: :uid
    first_name?: :string
    last_name?: :string
  return: :customer

# Broadcasts the current state of the Customer with given ID
#
# Broadcasts: customers#updated
#
customers/broadcast:
  params:
    id: :uid
  # no "return" here indicates this can only be invoked as COMMAND

# Returns a Customer with a given ID
#
customers/show:
  params:
    id: :uid
  return: :customer

# Returns a list of all existing Customers
#
customers/list:
  params:
  return:
    list:
      :array: :customer

# EVENTs produced by this resource
customers#created: :customer
customers#updated: :customer

```

## Note on pro's and con's

The benefit of the proposed approach is that, while allowing to design an API specification in a machine readable way, it keeps the specification concise
and *mostly* human readable.

The downside is that to produce a nicer human readable documentation
(e.g. HTML or PDF) out of these YAML specification files,
would probably require parsing the YAML and preserving
comments, which is not something that the common YAML parsers normally can do.

To summarize the properties of the proposed specification format:

* it is machine readable
* it is flexible enough to allow documenting any kind of Messaging API
* it is simple and concise enough to be *mostly* human readable

---
