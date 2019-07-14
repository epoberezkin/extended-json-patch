# extended-json-patch
Extended JSON Patch

This specification extends JSON Patch
([RFC6902](https://tools.ietf.org/html/rfc6902)).


## Motivation

- RFC6902 allows for object and array changes that are defined differently for
the same operation, but it doesn't provide a way to ensure that the value is an
array or an object, so it can lead to unpredictable results.
- Some operations require that a member exists for an operation to succeed, but
this requirement is implicit and can be confusing. Also, in some cases you may
want to just test the presense of a certain member without performing any action
with it.
- RFC6902 only allows replacing the whole string values - it is difficult to
understand what the change was with large multiline strings.


## Changes

- Type tests. Extended JSON Patch extends ["test" operation](#test) defined in
RFC6902 to allow testing for a target location type.
- Member existence test.
- Multi-line text editing. Extended JSON Patch defines `"edit"` operation that
allows to apply changes to large string value with a minimal size of the patch.


## Added and changed operations

### test

The "test" operation is extended: it can test one of the following:
- that a value at the target location is equal to a specified value, as defined
in RFC6902,
- that the value at the target location has a specified type,
- that the target location exists.

The operation object MAY contain a "value" member, as per RFC6902, or
the "type" member that conveys the type to be compared to the target location's
type.

If the operation object contains both "value" and "type" members the operation
MUST fail.


#### Testing value

If the operation object contains "value" member it behaves as defined in
RFC6902.


#### Testing existence

If the operation object contains neither "value" nor "type" members, the
operation tests for the existence of the target location.

The target location MUST exist for the operation to be considered successful.

Example:

```json
{ "op": "test", "path": "/a/b/c" }
```

#### Testing value type

If the operation object contains "type" value, then the target location type
MUST be equal to the "type" value for the operation to be considered successful.

The target location MUST exist for the operation to be considered successful.

The following types can be tested by this operation: "string", "number",
"array", "object", "integer", "boolean", "null".

- For the types "string", "number", "array" or "object" the target location
value MUST be, respectively, string, number, array or object for the operation
to succeed.
- For the type "integer", the target location value MUST be a number without a
fractional part or with a zero fractional part for the operation to succeed
(i.e. both 1 and 1.0 are considered to have "integer" type).
- For the type "boolean", the target location value MUST be a literal true or
false for the operation to succeed.
- For the type "null", the target location value MUST be a literal null for the
operation to succeed.

Example:

```json
{ "op": "test", "path": "/a/b/c", "type": "array" }
```

If the type test fails, the application of the entire patch document MUST fail
(as defined in [Error Handling](https://tools.ietf.org/html/rfc6902#section-5))
section of RFC6902.


### Text editing and testing operations

Example patch with all text operations:

```json
[
  {
    "op": "test",
    "path": "/a/b/c",
    "type": "string"
  },
  {
    "op": "add-text",
    "path": "/a/b/c",
    "pos": {"line": 1},
    "text": "Hello world\n"
  },
  {
    "op": "remove-text",
    "path": "/a/b/c",
    "pos": {"line": 1, "col": 7},
    "endPos": {"line": 1, "col": 12}
  },
  {
    "op": "replace-text",
    "path": "/a/b/c",
    "pos": {"line": 1, "col": 1},
    "endPos": {"line": 1, "col": 6},
    "text": "eyH"
  },
  {
    "op": "move-text",
    "from": "/a/b/c",
    "fromPos": {"line": 1, "col": 3},
    "fromEndPos": {"line": 1, "col": 4},
    "path": "/a/b/c",
    "pos": {"line": 1, "col": 1}
  },
  {
    "op": "copy-text",
    "from": "/a/b/c",
    "fromPos": {"line": 1, "col": 1},
    "fromEndPos": {"line": 1, "col": 4},
    "path": "/a/b/c",
    "pos": {"line": 1, "col": 5}
  },
  {
    "op": "test-text",
    "path": "/a/b/c",
    "pos": {"line": 1},
    "endPos": {"line": 2},
    "text": "Hey Hey"
  }
]
```
