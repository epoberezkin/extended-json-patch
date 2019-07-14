# extended-json-patch
Extended JSON Patch

This specification extends JSON Patch
([RFC6902](https://tools.ietf.org/html/rfc6902)).


# Abstract

Extended JSON Patch defines an extension to JSON Patch
[RFC6902](https://tools.ietf.org/html/rfc6902) to add support for
- testing target location existence and value type
- text editing operations to apply changes to specific parts of the string
values


## Table of Contents

- [Motivation](#motivation)
- [Changes](#changes)
- [Added and changed operations](#added-and-changed-operations)
  - [test](#test)
    - [Testing value](#testing-value)
    - [Testing existence](#testing-existence)
    - [Testing value type](#testing-value-type)
  - [Text editing and testing](#text-editing-and-testing)
    - [Position in text](#position-in-text)
    - [add-text](#add-text)
    - [remove-text](#remove-text)
    - [replace-text](#replace-text)
    - [move-text](#move-text)
    - [copy-text](#copy-text)
    - [test-text](#test-text)


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

- Type and location existence tests. Extended JSON Patch extends
["test" operation](#test) defined in RFC6902 to allow testing for a target
location type.
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


### Text editing and testing

Example patch with all text operations:

```json
[
  {
    "op": "test",
    "path": "/foo",
    "type": "string"
  },
  {
    "op": "add-text",
    "path": "/foo",
    "pos": {"line": 0},
    "text": "Hello there\n"
  },
  {
    "op": "remove-text",
    "path": "/foo",
    "pos": {"line": 0, "col": 6},
    "endPos": {"line": 0, "col": 11}
  },
  {
    "op": "replace-text",
    "path": "/foo",
    "pos": {"line": 0, "col": 0},
    "endPos": {"line": 0, "col": 5},
    "text": "eyH"
  },
  {
    "op": "move-text",
    "from": "/foo",
    "fromPos": {"index": 2},
    "fromEndPos": {"index": 3},
    "path": "/foo",
    "pos": {"index": 0}
  },
  {
    "op": "copy-text",
    "from": "/foo",
    "fromPos": {"line": 0, "col": 0},
    "fromEndPos": {"line": 0, "col": 3},
    "path": "/foo",
    "pos": {"line": 0, "col": 4}
  },
  {
    "op": "test-text",
    "path": "/foo",
    "pos": {"line": 0},
    "endPos": {"line": 1},
    "text": "Hey Hey"
  }
]
```


#### Position in text

Text editing operations use objects that define the position in text. A position
object MUST be one of the following:

- it MUST contain a member "index" that designates a zero-based index of the
character in the string. All characters are considered occupying exactly one
index regardless how many bytes they use and how many columns they occupy.
- it MUST contain a member "line" with an optional member "column" that
designate a zero-based line and column numbers of the position in the text.
"column" member defaults to 0 if absent.

If position object contains any of the following combindations of members the
operation MUST fail:

- "index" and "line"
- "index" and "column"
- neither "index" nor "line" member

Whitespace:

- the only character that increases line number is line feed ('\n'), so if your
JSON string has '\r\n' sequence, it will still be counted as one line,
- both '\r' and '\n' are counted as a character when determining index (it is
possible to slice sections of JSON string using index property), but column
counter is reset when '\r' or '\n' is encountered,
- tabs ('\t') are counted as four spaces when determining column but as a
single character for index. The PATCH request may specify a different tab size.

The text position is considerent existing in the string if the number of
lines in the string is greater or equal than "line" member (if present), and
the number of columns in the specified "line" is greater or equal than "column"
member (if present), and the number of characters in the string is greater or
equal than "index" member (if present).

Text positions range is defined as the array of two text positions. The range
is considered valid if both text positions exist in the string and the second
text position is later in the text that the first.


#### add-text

The "add-text" operation adds text to a target position in the string in the
target location.

The operation object MUST contain:
- a "text" member with a string value whose content specifies the text to be
added inside a string at a targer location.
- a "pos" member with a text position object whose contents specify the text
position where the text from "text" member will be added as a result of the
operation.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value, and the text position MUST exist in this string (see
[Position in text](#position-in-text)).

Example:

```json
{
  "op": "add-text",
  "path": "/foo",
  "pos": {"line": 0},
  "text": "Hello there\n"
}
```

If the value of the object modified with the patch was `{"foo": "Welcome!"}`,
it will become `{"foo": "Hello there\nWelcome!"}` after the patch is applied.


#### remove-text

The "remove-text" operation removes the text from the string at the target
location between target text positions.

The operation object MUST contain a "pos" and "endPos" members with a text
position objects whose contents specify the text positions between which the 
text will be removed from the string.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value, and the text position range ["pos", "endPos"] MUST be valid (see
[Position in text](#position-in-text)).

Example:

```json
{
  "op": "remove-text",
  "path": "/foo",
  "pos": {"line": 0, "col": 6},
  "endPos": {"line": 0, "col": 11}
}
```

If the value of the object modified with the patch was
`{"foo": "Hello there\nWelcome!"}`, it will become
`{"foo": "Hello \nWelcome!"}` after the patch is applied.


#### replace-text

The "remove-text" operation replaces the text in the string at the target
location between target text positions with a new text.

The operation object MUST contain:
- a "text" member with a string value whose content specifies the replacement
text.
- a "pos" and "endPos" members with a text position objects whose contents
specify the text positions between which the text will be replaced in the
string with a new text.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value, and the text position range ["pos", "endPos"] MUST be valid (see
[Position in text](#position-in-text)).

```json
{
  "op": "replace-text",
  "path": "/foo",
  "pos": {"line": 0, "col": 0},
  "endPos": {"line": 0, "col": 5},
  "text": "eyH"
}
```

This operation is functionally identical to a "remove-text" operation for a
text, followed immediately by an "add-text" operation at the same location with
the replacement text.

If the value of the object modified with the patch was
`{"foo": "Hello \nWelcome!"}`, it will become `{"foo": "eyH \nWelcome!"}` after
the patch is applied.


#### move-text

The "move-text" operation removes the text from the string at the specified
location between specified text positions and adds it to a target position in
the string in the target location.

The operation object MUST contain:
- a "from" member, which is a string containing a JSON Pointer value that
references the location in the target document to move the text from.
- a "fromPos" and "fromEndPos" members with a text position objects whose
contents specify the text positions between which the text will be removed to
be added to a target location.
- a "pos" member with a text position object whose contents specify the text
position where the text from "text" member will be added as a result of the
operation.

For the operation to succeed:
- the specified "from" and target ("path") locations MUST exist
- both "from" and target locations MUST contain string values
- the text position range ["fromPos" and "fromEndPos"] MUST be a valid range in
the string at "from" location (see [Position in text](#position-in-text)).
- target text postion "pos" MUST exist in the string at the target location.

If the "from" location is the same as the target location, the text is
moved withing the same string. In this case the target text position defines
the position in the string after the text was removed and this position MUST
exist in the modified string.

Example:

```json
{
  "op": "move-text",
  "from": "/foo",
  "fromPos": {"index": 2},
  "fromEndPos": {"index": 3},
  "path": "/foo",
  "pos": {"index": 0}
}
```

This operation is functionally identical to a "remove-text" operation on the
"from" location with the text positions specified by "fromPos" and "fromEndPos"
followed immediately by an "add-text" operation at the target location and the
text position with the text that was just removed.

If the value of the object modified with the patch was
`{"foo": "eyH \nWelcome!"}`, it will become `{"foo": "Hey \nWelcome!"}` after
the patch is applied.


#### copy-text

The "copy-text" operation copies the text in the string at the specified
location between specified text positions to the target position in the string
in the target location.

The operation object MUST contain:
- a "from" member, which is a string containing a JSON Pointer value that
references the location in the target document to move the text from.
- a "fromPos" and "fromEndPos" members with a text position objects whose
contents specify the text positions between which the text will be removed to
be added to a target location.
- a "pos" member with a text position object whose contents specify the text
position where the text from "text" member will be added as a result of the
operation.

For the operation to succeed:
- the specified "from" and target ("path") locations MUST exist
- both "from" and target locations MUST contain string values
- the text position range ["fromPos" and "fromEndPos"] MUST be a valid range in
the string at "from" location (see [Position in text](#position-in-text)).
- target text postion "pos" MUST exist in the string at the target location.

If the "from" location is the same as the target location, the text is
copied withing the same string. The target text position MAY be between the
specified positions to copy the text from.

Example:

```json
{
  "op": "copy-text",
  "from": "/foo",
  "fromPos": {"line": 0, "col": 0},
  "fromEndPos": {"line": 0, "col": 3},
  "path": "/foo",
  "pos": {"line": 0, "col": 4}
}
```

This operation is functionally identical to an "add-text" operation at the
target location and the text position with the text that was copied from the
specified location and positions in the "from", "fromPos" and "fromEndPos"
members.

If the value of the object modified with the patch was
`{"foo": "Hey \nWelcome!"}`, it will become `{"foo": "Hey Hey\nWelcome!"}` after
the patch is applied.


#### test-text

The "text-text" operation can be used to test the equality of the text; to test
the existence of text position and the validity of the text position range.

##### Testing text equality

The "test-text" operation can be used to test that a text at the target
location and the text position is equal to a specified text.

In this case the operation object MUST contain:
- a "text" member with a string value whose content specifies the replacement
text.
- a "pos" and "endPos" members with a text position objects whose contents
specify the text positions between which the text will be replaced in the
string with a new text.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value, the text position range ["pos", "endPos"] MUST be valid (see
[Position in text](#position-in-text)) and the text between specified positions
MUST be equal to the specified text.

Example:

```json
{
  "op": "test-text",
  "path": "/foo",
  "pos": {"line": 0},
  "endPos": {"line": 1},
  "text": "Hey Hey"
}
```

If the value of the object to which the patch is applied is
`{"foo": "Hey Hey\nWelcome!"}`, the operation will succeed.


##### Testing a text position

The "test-text" operation can be used to test that the target text position
exists in a string at the target location.

In this case the operation object MUST contain a "pos" member with a text
position object.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value and the text position MUST exist in this string (see
[Position in text](#position-in-text)).

Example:

```json
{
  "op": "test-text",
  "path": "/foo",
  "pos": {"line": 1}
}
```

If the value of the object to which the patch is applied is
`{"foo": "Hey Hey\nWelcome!"}`, the operation will succeed.

If the value of the object to which the patch is applied is
`{"foo": "Hey Hey"}`, the operation will fail because the string has only one
line.


##### Testing a text positions range

The "test-text" operation can be used to test that the target text positions
range exists in a string at the target location.

In this case the operation object MUST contain a "pos" and "endPos" members
with a text position objects whose contents specify the text positions between
which the text will be replaced in the string with a new text.

For the operation to succeed, the target location MUST exist, it MUST contain a
string value, the text position range ["pos", "endPos"] MUST be valid (see
[Position in text](#position-in-text)) and the "endPos" text position should be
later in the text than "pos" text position.

Example:

```json
{
  "op": "test-text",
  "path": "/foo",
  "pos": {"line": 0},
  "endPos": {"line": 1}
}
```

If the value of the object to which the patch is applied is
`{"foo": "Hey Hey\nWelcome!"}`, the operation will succeed.

If the value of the object to which the patch is applied is
`{"foo": "Hey Hey"}`, the operation will fail because the string has only one
line.
