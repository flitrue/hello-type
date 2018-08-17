HelloType
=========

An ECMAScript data type check library.

## Install

```
npm i -S hello-type
```

## Usage

```js
import HelloType, { Dict, Enum, Tuple, List, Type, Rule, Self, IfExists } from 'hello-type'
```

or

```js
const { Dict, Enum, Tuple, List, Type, Rule, Self, HelloType, IfExists } = require('hello-type')
```

## Type

To define a type of data, you can use `Type`. It is a class.

```js
const PersonType = new Type({
  name: String,
  age: Number,
  male: Boolean,
  body: {
    head: HeadType, // HeadType is another custom type made of `Type`
    neck: NeckType,
    foot: FootType,
  },
  friends: [FriendType],
  familyCount: Enum(3, 4, 5), // use `Enum()` to make a Enum type
})
```

Then use this data type to check a data:

```js
function welcome(person) {
  PersonType.assert(person) // If the param `person` not match PersonType, an error will be thrown out
}
```

Rules:

- String: should be a string
- Number: should be a finite number, `NaN` `"123"` and `Finite` will not match
- Boolean: should be one of `true` or `false`
- Object: should be a normal object like `{}`, instance of class, array and Object self will not match
- Array: should be an array
- Function: should be a function
- RegExp: should be a regexp
- ... any other js data prototype
- Dict(?): should be structed with passed value
- List(?, ?, ?): should be an array which has certain sturcted item
- Enum(?, ?, ?): should be one of these values
- Tuple(?, ?, ?): should be same number and structure with each value, always used for function parameters
- ?: any value to equal, i.e. new Type({ name: 'tomy' }), name should must be 'tomy'
- ?: an instance of `Type`, will flow the rules of it
- new Rule(factory): a custom rule
- Any, Self

A type instance have members:

**assert(...args)**

Assert whether the args match the type.
If not match, it will use `throw` to break the program.
Return undefined.

**test(...args)**

Assert whether the args match the type.
Return true if match, return false if not match.

**catch(...args)**

Assert whether the args match the type.
Return null if match, and return error object if not match.

```js
let error = PersonType.catch(person)
```

If there is no error, `null` will be returned.

**trace(...args).with(fn)**

Assert whether the args match the type.
It will run async.
If not match, `fn` will run. You can do like:

```js
PersonType.trace(person).with((error) => console.log(error))
```

`fn` has three parameters:

- error: the catched error, if pass, it will be undefined
- args: array, targets to match
- type: which type be used to match

```js
PersonType.trace(person).with((error, [person], type) => {
  if (error) {
    console.log(person, 'not match', type, 'error:', error)
  }
})
```

It will return a resolved promise anyway:

```js
let error = await PersonType.trace(person).with(fn)
```

**strict/Strict**

Whether use strict mode, default mode is false. If you use strict mode, object properties count should match, array length should match, even you use `IfExists`.

```js
const MyType = new Type({
  name: String,
  age: Number,
})
MyType.Strict.assert({
  name: 'tomy',
  age: 10,
  height: 172, // this property is not defined in type, so assert will throw an error
})
```

```js
const MyType = new Type([Number, Number])
MyType.Strict.assert([1]) // array length should be 2, but here just passed only one
```

## Dict

A Dict is a type of object which has only one level properties.

- @type function
- @param object
- @return an instance of `Type`

```js
const DictType = Dict({
  name: String,
  age: Number,
})
```

You can pass nested object, but are recommended to use another Dict instead:

```js
const BodyType = Dict({
  head: Object,
  foot: Object,
})
const PersonType = Dict({
  name: String,
  age: Number,
  body: BodyType,
})
```

_What's the difference between Dict and Object?_

An Object match any structure of object. However, a Dict match certain structure of object.

_What's the difference between Dict and new Type?_

- _Dict_ receive only one parameter. 
- _Dict_ only receive object, _new Type_ receive any.
- If you pass only one object into _new Type_, they are the same.

## List

A list is an array in which each item has certain structure.

- @type function
- @param array
- @return an instance of `Type`

```js
const ListType = List([String, Number])
```

The pending verify array's items should be right data type as the given order. 
If the array length is longer than the rule's length, the overflow ones should be one of these rules. 
For example, the 3th item should be Enum(String, Number):

```js
ListType.test(['string', 1, 'str']) // true
ListType.test(['string', 1, 2]) // true
ListType.test(['string', 1, {}]) // false
```

_What's the difference between List and Array?_

An Array match any structure for its item. However, a List match certain structure for each item.

_What's the difference between List and Tuple?_

Tuple has no relation with array, it is a group of scattered items with certain order.

## Tuple

A tuple is a group of scattered items with certain order, the length of tuple can not be changed, each item can have different structure.

- @type function
- @params any
- @return an instance of `Type`

```js
const ParamsType = Tuple(Object, Number, String)
ParamsType.assert({}, 1, 'ok')
```

_What's the difference between Tuple and new Type?_

Tuple can use `IfExists` at the end of parameters, so that you can assert less arguments. However _new Type_ assert method should must receive given count of arguments:

```js
const ParamsType = Tuple(Object, IfExists(Number), IfExists(String))
ParamsType.test({}, 1) // true

const SomeType = Tuple(Object, IfExists(Number), IfExists(String))
SomeType.test({}, 1) // false, arguments length not match
```

## Enum

A enum is a set of values from which the given value should pick.

- @type function
- @params any
- @return an instance of `Type`


```js
const ColorType = Enum('red', 'white', 'green')
ColorType.test('black') // false
```

```js
const ColorType = Enum(String, Number)
ColorType.test('black') // true
ColorType.test(2) // true
ColorType.test([]) // false
```

## Range

A range is a threshold of numbers. The given value should be a nunmber and in the range.

- @type function
- @params number, only two
- @return an instance of `Type`

```js
const RangeType = Range(0, 1)
RangeType.test(0.5) // true
RangeType.test(3) // false
RangeType.test(0) // true, 0 and 1 is in range
```

## Rule

Create a custom rule:

```js
const CustomRule = new Rule(function(value) {
  if (value !== 'ok') {
    return new Error(value + ' not equal `ok`')
  }
})
const CustomType = new Type(CustomRule)
CustomType.test('ok') // true
```

The function which you passed into `new Rule()` should have a parameter.
If you want to make assert fail, you should must return an instance of Error.

Notice: CustomRule is just a instance of Rule, it is not a type, do not have `assert` `trace` and so on.

**Any**

There is a special rule called `Any`, it means your given value can be any type:

```js
import { Dict, Any } from 'hello-type'

const MyType = Dict({
  top: Any,
})
```

**IfExists**

Only when the value exists will the rule works.
If there is no value, or the value is undefined, this rule can be ignored.

- @type function
- @param any rule
- @return instance of Type/Rule

```js
import { Dict, IfExists } from 'hello-type'

const PersonType = Dict({
  name: String,
  age: IfExists(Number),
})
PersonType.test({ name: 'tomy' }) // true
PersonType.test({ name: 'tomy', age: 10 }) // true
PersonType.test({ name: 'tomy', age: null }) // false
```

If there is `age` property, PersonType will check its value.
If `age` property does not exist, the checking will be ignored.

This rule will work in strict mode, too!

```js
PersonType.Strict.test({ name: 'tomy' }) // false
```

In strict mode, IfExists will be ignored, you must pass certain type of data to assert.

`IfExists` only works for Dict, List and Tuple.

```js
const PersonType = Dict({
  name: String,
  children: [IfExists(Object)], // => can be '[]' or '[{...}, ...]'
})
```

```js
const ParamsType = Tuple(String, IfExists(Number)) // => can be ('name') or ('name', 10)
```

In Tuple, only the rest items can be if_exists.

## HelloType

The `HelloType` is a set of methods to use type assertions more unified.

**expect.toBe.typeof**

```js
HelloType.expect(SomeType).toBe.typeof(someobject) // it is the same as `SomeType.assert(someobject)`
```

**is.typeof**

```js
let bool = HelloType.is(SomeType).typeof(someobject)
```

**catch.by**

```js
let error = HelloType.catch(someobject).by(SomeType) // it is the same as `SomeType.catch(someobject)`
```

**trace.by.with**

```js
HelloType.trace(someobject).by(SomeType.Strict).with((error, [args], type) => { 
  // strict mode
  // it is the same as `SomeType.Strict.trace(someobject).with(fn)`
  // ...
}) 
```

**decorate**

Use to decorate class and its members:

```js
@HelloType.decorate.with((arg) => SomeType.assert(arg))
class SomeClass {}
```

## Test

```
npm test
```

## MIT License

Copyright 2018 tangshuang

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.