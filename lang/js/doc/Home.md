<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->


This page is meant to provide a brief overview of `avro`'s API:

+ [What is a `Type`?](#what-is-a-type)
+ [How do I get a `Type`?](#how-do-i-get-a-type)
+ [What about Avro files?](#what-about-avro-files)
+ [Next steps](#next-steps)


## What is a `Type`?

Each Avro type maps to a corresponding JavaScript [`Type`](API#class-type):

+ `int` maps to `IntType`.
+ `array`s map to `ArrayType`s.
+ `record`s map to `RecordType`s.
+ etc.

An instance of a `Type` knows how to [`decode`](Api#typedecodebuf-pos-resolver)
and [`encode`](Api#typeencodeval-buf-pos) and its corresponding objects. For
example the `StringType` knows how to handle JavaScript strings:

```javascript
var stringType = new avro.types.StringType();
var buf = stringType.toBuffer('Hi'); // Buffer containing 'Hi''s Avro encoding.
var str = stringType.fromBuffer(buf); // === 'Hi'
```

The [`toBuffer`](API#typetobufferval) and
[`fromBuffer`](API#typefrombufferval-resolver-nocheck) methods above are
convenience functions which encode and decode a single object into/from a
standalone buffer.

Each `type` also provides other methods which can be useful. Here are a few
(refer to the [API documentation](API#avro-types) for the full list):

+ JSON-encoding:

  ```javascript
  var jsonString = type.toString('Hi'); // === '"Hi"'
  var str = type.fromString(jsonString); // === 'Hi'
  ```

+ Validity checks:

  ```javascript
  var b1 = stringType.isValid('hello'); // === true ('hello' is a valid string.)
  var b2 = stringType.isValid(-2); // === false (-2 is not.)
  ```

+ Random object generation:

  ```javascript
  var s = stringType.random(); // A random string.
  ```


## How do I get a `Type`?

It is possible to instantiate types directly by calling their constructors
(available in the `avro.types` namespace; this is what we used earlier), but in
the vast majority of use-cases they will be automatically generated by parsing
an existing schema.

`avro` exposes a [`parse`](Api#parseschema-opts) method to do the
heavy lifting:

```javascript
// Equivalent to what we did earlier.
var stringType = avro.parse({type: 'string'});

// A slightly more complex type.
var mapType = avro.parse({type: 'map', values: 'long'});

// The sky is the limit!
var personType = avro.parse({
  name: 'Person',
  type: 'record',
  fields: [
    {name: 'name', type: 'string'},
    {name: 'phone', type: ['null', 'string'], default: null},
    {name: 'address', type: {
      name: 'Address',
      type: 'record',
      fields: [
        {name: 'city', type: 'string'},
        {name: 'zip', type: 'int'}
      ]
    }}
  ]
});
```

Of course, all the `type` methods are available. For example:

```javascript
personType.isValid({
  name: 'Ann',
  phone: null,
  address: {city: 'Cambridge', zip: 02139}
}); // === true

personType.isValid({
  name: 'Bob',
  phone: {string: '617-000-1234'},
  address: {city: 'Boston'}
}); // === false (Missing the zip code.)
```

Since schemas are often stored in separate files, passing a path to `parse`
will attempt to load a JSON-serialized schema from there:

```javascript
var couponType = avro.parse('./Coupon.avsc');
```

For advanced use-cases, `parse` also has a few options which are detailed the
API documentation.


## What about Avro files?

Avro files (meaning [Avro object container files][object-container]) hold
serialized Avro records along with their schema. Reading them is as simple as
calling [`createFileDecoder`](Api#createfiledecoderpath-opts):

```javascript
var personStream = avro.createFileDecoder('./persons.avro');
```

`personStream` is a [readable stream][rstream] of decoded records, which we can
for example use as follows:

```javascript
personStream.on('data', function (person) {
  if (person.address.city === 'San Francisco') {
    doSomethingWith(person);
  }
});
```

In case we need the records' `type` or the file's codec, they are available by
listening to the `'metadata'` event:

```javascript
personStream.on('metadata', function (type, codec) { /* Something useful. */ });
```

To access a file's header synchronously, there also exists an
[`extractFileHeader`](Api#extractfileheaderpath-opts) method:

```javascript
var header = avro.extractFileHeader('persons.avro');
```

Writing to an Avro container file is possible using
[`createFileEncoder`](Api#createfileencoderpath-type-opts):

```javascript
var encoder = avro.createFileEncoder('./processed.avro', type);
```


## Next steps

The [API documentation](Api) provides a comprehensive list of available
functions and their options. The [Advanced usage section](Advanced-usage) goes
through a few examples to show how the API can be used.



[object-container]: https://avro.apache.org/docs/current/spec.html#Object+Container+Files
[rstream]: https://nodejs.org/api/stream.html#stream_class_stream_readable
