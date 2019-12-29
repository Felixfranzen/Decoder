# Decoder

A powerful, well tested, data decoder for Typescript.

## Install

Simply run

```
npm i elm-decoders
```

Or

```
yarn add elm-decoders
```

Then at the top of your file add:

```
import {Decoder} from 'elm-decoders'
```

## Table of Content

- [Decoder](#decoder)
  - [Install](#install)
  - [Motivation](#motivation)
  - [Tutorial](#tutorial)
  - [API Docs Methods:](#api-docs-methods-)
    - [`map = <S>(mapFunction: (arg: T) => S): Decoder<S>`](#-map----s--mapfunction---arg--t-----s---decoder-s--)
    - [`nullable = (): Decoder<T | null>`](#-nullable-------decoder-t---null--)
    - [`withDefault = <E>(value: T | E) => Decoder<T | E>`](#-withdefault----e--value--t---e-----decoder-t---e--)
    - [`satisfy = ({predicate, failureMessage,}: {predicate: (arg: T) => boolean; failureMessage?: (attemptedData: any) => string;}): Decoder<T>`](#-satisfy-----predicate--failuremessage-----predicate---arg--t-----boolean--failuremessage----attempteddata--any-----string-----decoder-t--)
    - [`or = <S>(decoder: Decoder<S>): Decoder<T | S>`](#-or----s--decoder--decoder-s----decoder-t---s--)
    - [`run = (data: any): {type: "OK": value: T} | {type: "FAIL", error: string}`](#-run----data--any----type---ok---value--t-----type---fail---error--string--)
    - [`andThen = <S>(dependentDecoder: (res: T) => Decoder<S>): Decoder<S>`](#-andthen----s--dependentdecoder---res--t-----decoder-s----decoder-s--)
    - [API Static methods](#api-static-methods)
    - [`number: Decoder<number>`](#-number--decoder-number--)
    - [`literalString = <T extends string>(str: T): Decoder<T>`](#-literalstring----t-extends-string--str--t---decoder-t--)
    - [`literalNumber = <T extends number>(number: T): Decoder<T>`](#-literalnumber----t-extends-number--number--t---decoder-t--)
    - [`string: Decoder<string> = new Decoder((data: any)`](#-string--decoder-string----new-decoder--data--any--)
    - [`fail = <T>(message: string): Decoder<T>`](#-fail----t--message--string---decoder-t--)
    - [`ok = <T>(value: T): Decoder<T>`](#-ok----t--value--t---decoder-t--)
    - [`array = <T>(decoder: Decoder<T>): Decoder<T[]>`](#-array----t--decoder--decoder-t----decoder-t----)
    - [`boolean: Decoder<boolean>`](#-boolean--decoder-boolean--)
    - [`field = <T>(fieldName: string, decoder: Decoder<T>): Decoder<T>`](#-field----t--fieldname--string--decoder--decoder-t----decoder-t--)
    - [`object = <T>(object: { [P in keyof T]: Decoder<T[P]> }): Decoder<T>`](#-object----t--object-----p-in-keyof-t---decoder-t-p-------decoder-t--)
  - [Credit](#credit)
  - [Alternatives](#alternatives)
  - [Local Development](#local-development)
    - [`npm start` or `yarn start`](#-npm-start--or--yarn-start-)
    - [`npm run build` or `yarn build`](#-npm-run-build--or--yarn-build-)
    - [`npm test` or `yarn test`](#-npm-test--or--yarn-test-)

## Motivation

When using Typescript, there is very little we can do to guarantee that the
incoming data is correct. This means that errors can occurr anywhere in the
code, introducing odd errors and unexpected behaviour with no way of knowing
where the error came from. By instead validating our data at the start (for
example when receiving an incoming request), we can handle the error early
and give better error messages to the developer. This creates a better
developer experience and making us more confident and productive. It gives us
stronger guarantees.

Another benefit of using Decoders is that you can pick the best data model
for your problem and convert all incoming data sources to fit that model.
This makes it easier to write business logic separately from the acquasition
of the data. Also, by picking the right data model for your problem helps with bug
reduction and maintainability.

Decoders are great for validating and converting data from various sources:
Kafka data, request bodies or APIs to name a few examples.

## Tutorial

Decoder provides us with a few primitive decoders and a few methods to craft
new ones. Let's say we have the following data:

```
const incomingData: any = {
    name: "Nick",
    age: 30
}
```

And we have an interface `User`:

```
interface User {
    name: string
    age: number
}
```

To validate that `incomingData` is a `User`, Decoder provides an `object` primitive.

```
import {Decoder} from 'elm-decoder'

const userDecoder: Decoder<User> = Decoder.object({
    name: Decoder.string,
    age: Decoder.number
})
```

Now we can validate `incomingData` and ensure the data is correct.

```
const result = userDecoder.run(incomingData)
```

`run` returns a [Discriminated
union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions),
meaning is returns _either_ `{type: "OK", value: T}` _or_ `{type: "FAIL": error: string}`. This means that we are forced to check if the data received is correct or contains an error. Doing so
is as simple as a switch case:

```
switch(result.type) {
    case "OK":
        doUserThing(result.value)
    case "FAIL":
        handleError(result.error)
}
```

Decoder also provides a few methods for creating new decoders. For example, if
we want to create a set decoder, we can use the `map` method.

```
const intSetDecoder: Decoder<Set<number>> = Decoder.array(Decoder.number).map(numberArray => new Set(numberArray))
```

This was a brief introduction. From here, please check the API documentation
to find out more what you can do and try it for yourself!

## API Docs Methods:

### `map = <S>(mapFunction: (arg: T) => S): Decoder<S>`

Transform a decoder. Useful for narrowing data types. For example:

```
const setDecoder: Decoder<Set<number>> =
     Decoder.array(Decoder.number).map(numberArray => new Set(numberArray))
```

### `nullable = (): Decoder<T | null>`

Turns a decoder into a nullable one. For example:

```
const optionalNumber = Decoder.number.nullable()

optionalNumber.run(null) // OK, value is null
optionalNumber.run(undefined) // OK, value is null
optionalNumber.run(5) // OK, value is 5
optionalNumber.run('hi') // FAIL
```

### `withDefault = <E>(value: T | E) => Decoder<T | E>`

Sets a default value to the decoder if it fails.

```
const nrDecoder = Decoder.number.withDefault(0)

nrDecoder.run(5) // OK 5
nrDecoder.run('hi') // OK 0
```

### `satisfy = ({predicate, failureMessage,}: {predicate: (arg: T) => boolean; failureMessage?: (attemptedData: any) => string;}): Decoder<T>`

Add an extra predicate to the decoder.

Example:

```
const naturalNumber = Decoder.number.satisfy({
 predicate: n => n>0
 failureMessage: data => `data is not a natural number > 0, it is: ${data}`
})
naturalNumber.run(5) // OK, 5
naturalNumber.run(-1) // FAIL
```

### `or = <S>(decoder: Decoder<S>): Decoder<T | S>`

Attempt two decoders.

Example:

```
type Names = "Jack" | "Sofia"
const enums: Decoder<Names> = Decoder.literal("Jack").or(Decoder.literal("Sofia"))

enums.run("Jack") // OK
enums.run("Sofia") // OK
enums.run("Josefine") // Fail
```

### `run = (data: any): {type: "OK": value: T} | {type: "FAIL", error: string}`

Run a decoder on data. The result is a [discriminated union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions). In order to
access the result you must explicitly handle the failure case with a switch.

Example:

```
const userCredentials: Decoder<Credentials> = Decoder.object({...})

//... Somewhere else
const result = userCredentials.run(request.query)
switch(result.type) {
    case "OK":
        login(result.value)
    case "FAIL":
        throw new Error(result.error)
}
```

### `andThen = <S>(dependentDecoder: (res: T) => Decoder<S>): Decoder<S>`

Create decoders that are dependent on previous results.

Example:

```
const version = Decoder.object({
 version: Decoder.literalNumber(0).or(Decoder.literalNumber(1)),
});
const api = ({ version }: { version: 0 | 1 }): Decoder<{...}> => {
  switch (version) {
     case 0:
       return myFirstDecoder;
     case 1:
       return mySecondDecoder;
}
};
const versionedApi = version.andThen(api);
```

### API Static methods

### `number: Decoder<number>`

Parses data and casts it to number.
It will succeed on '5' and 5

### `literalString = <T extends string>(str: T): Decoder<T>`

Decodes the exact string and sets it to a string literal type. Useful for
parsing (discriminated) unions.

```
type Names = "Jack" | "Sofia"
const enums: Decoder<Names> = Decoder.literalString("Jack").or(Decoder.literalString("Sofia"))
```

### `literalNumber = <T extends number>(number: T): Decoder<T>`

Decodes the exact number and sets it to a number literal type. Useful for
parsing (discriminated) unions.

```
type Versions = Decoder<1 | 2>
const versionDecoder: Decoder<Versions> = Decoder.literalNumber(1).or(Decoder.literalNumber(2))
```

### `string: Decoder<string> = new Decoder((data: any)`

Decodes a string.

```
Decoder.string.run('hi') // OK
Decoder.string.run(5) // Fail
```

### `fail = <T>(message: string): Decoder<T>`

Create a decoder that always fails, useful in conjunction with andThen.

### `ok = <T>(value: T): Decoder<T>`

Create a decoder that always suceeds, useful in conjunction with andThen

### `array = <T>(decoder: Decoder<T>): Decoder<T[]>`

Create an array decoder given a decoder for the elements.

```
Decoder.array(Decoder.string).run(['hello','world']) // OK
Decoder.array(Decoder.string).run(5) // Fail
```

### `boolean: Decoder<boolean>`

A decoder for booleans.

```
Decoder.boolean.run(true) // succeeds
Decoder.boolean.run(1) // fails
```

### `field = <T>(fieldName: string, decoder: Decoder<T>): Decoder<T>`

Decode a specific field in an object using a given decoder.

```
const versionDecoder = Decoder.field("version", Decoder.number)

versionDecoder.run({version: 5}) // OK
versionDecoder.run({name: "hi"}) // fail
```

### `object = <T>(object: { [P in keyof T]: Decoder<T[P]> }): Decoder<T>`

Create a decoder for an object T. Will currently go through each field and
collect all the errors but in the future it might fail on first.

object is a [Mapped type](https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types),
an object containing only decoders for each field. For example
`{name: Decoder.string}` is a valid `object`.

```
interface User {
   name: string
   email: string
}

// typechecks
const decodeUser<User> = Decoder.object({name: Decoder.string, email: Decoder.email})

// will not typecheck, object must have the same field names as user and
// only contain decoders.
const decodeUser<User> = Decoder.object({nem: 'Hi'})
```

## Credit

This library is essentially a rewrite of Nvie's (decoders.js)[https://github.com/nvie/decoders]
with some small changes. decoders.js is inspired by Elm's decoders.

## Alternatives

- (Joi)[https://github.com/hapijs/joi]. The currently most popular validator for data.
  While it is useful for Javascript, it's Typescript support is lackluster as
  it has no way of ensuring that a validator actually adheres to a certain type.
  This means that on top of writing the validator, you will have to also manually
  write unit tests to ensure that your validator adheres to your type or interface.
  This creates way too much boilerplate, or relies on the developer to not make mistakes
  which defeats the purpose of having static types in the first place
- (decoders.js)[https://github.com/nvie/decoders]. This library is essentially a
  carbon copy with a few changes. For one, this library supports parsing literals
  and secondly this library features a dot syntax for operations such as `map`,
  `andThen`.

## Local Development

Below is a list of commands you will probably find useful.

### `npm start` or `yarn start`

Runs the project in development/watch mode. Your project will be rebuilt upon changes. TSDX has a special logger for you convenience. Error messages are pretty printed and formatted for compatibility VS Code's Problems tab.

<img src="https://user-images.githubusercontent.com/4060187/52168303-574d3a00-26f6-11e9-9f3b-71dbec9ebfcb.gif" width="600" />

Your library will be rebuilt if you make edits.

### `npm run build` or `yarn build`

Bundles the package to the `dist` folder.
The package is optimized and bundled with Rollup into multiple formats (CommonJS, UMD, and ES Module).

<img src="https://user-images.githubusercontent.com/4060187/52168322-a98e5b00-26f6-11e9-8cf6-222d716b75ef.gif" width="600" />

### `npm test` or `yarn test`

Runs the test watcher (Jest) in an interactive mode.
By default, runs tests related to files changed since the last commit.
