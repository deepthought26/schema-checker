# schema-checker

[![npm version](https://img.shields.io/npm/v/schema-checker.svg)](https://www.npmjs.com/package/schema-checker)
[![Build Status](https://github.com/deepthought26/schema-checker/actions/workflows/build.yml/badge.svg)](https://github.com/deepthought26/schema-checker/actions/workflows/build.yml)
[![License: MIT](https://img.shields.io/npm/l/schema-checker.svg)](LICENSE)

**Runtime validation and type inference for TypeScript.**

`schema-checker` is a strongly typed validation library that models data shapes with function-based validators. Schemas validate at runtime and infer accurate TypeScript types at compile time, so you define your data contract once and use it in both places.

Previously published as [Funval](https://www.npmjs.com/package/funval).

## Why schema-checker?

| schema-checker | Joi (example) |
| --- | --- |
| Types inferred from schemas | Types defined separately |
| Native TypeScript-style validators (`string`, `number`, `array`) | Joi-specific API |
| Composable validator chains | Chainable, but separate from TS types |
| Zero runtime dependencies | External dependency tree |

```ts
// schema-checker
const UserSchema = Schema({
  name: string,
  amount: number,
  flags: array.of(string).optional(),
});

type User = Type<typeof UserSchema>;
```

```ts
// Joi equivalent
const UserSchema = Joi.object({
  name: Joi.string().required(),
  amount: Joi.number().required(),
  flags: Joi.array().items(Joi.string()),
});

type User = {
  name: string;
  amount: number;
  flags?: string[];
};
```

## Features

- **Readable schemas** — Validators mirror TypeScript primitives (`string`, `number`, `boolean`, `array`, `unknown`, and more).
- **Less duplication** — Reuse and compose validators to build new types quickly.
- **Compile-time safety** — TypeScript catches invalid schema usage before runtime.
- **Composable chains** — Combine validators to transform and validate data in one pipeline.
- **Sync and async** — Promise-returning validators are detected automatically.
- **Zero dependencies** — Small runtime footprint.
- **Plain JavaScript** — Works in projects with or without TypeScript.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Creating Custom Validators](#creating-custom-validators)
- [Validator Chains](#validator-chains)
- [API Reference](#api-reference)
- [License](#license)

## Installation

```bash
npm install schema-checker
```

Requires Node.js 12 or later.

## Quick Start

```ts
import Schema, { Type, string, number, array } from 'schema-checker';

const UserSchema = Schema({
  name: string.trim().normalize().between(3, 40).optional(),
  username: /^[a-z0-9]{3,10}$/,
  status: Schema.either('active' as const, 'suspended' as const),
  items: array
    .of({
      id: string,
      amount: number.gte(1).integer(),
    })
    .min(1),
});

type User = Type<typeof UserSchema>;
const validator = UserSchema.destruct();

const [err, user] = validator({
  username: 'john1',
  // TypeScript error: '"unregistered"' is not assignable to '"active" | "suspended"'
  status: 'unregistered',
  items: [{ id: 'item-1', amount: 20 }],
});

console.log(err);
// ValidationError: status: Expect value to equal "suspended"
```

## Creating Custom Validators

Define a function that accepts `unknown`, validates the input, and returns the typed value:

```ts
import * as EmailValidator from 'email-validator';

function Email(input: unknown): string {
  if (!EmailValidator.validate(String(input))) {
    throw new TypeError(`Invalid email address: "${input}"`);
  }

  return input as string;
}
```

Use it in a schema:

```ts
const UserSchema = Schema({
  email: Email,
});
```

For optional fields, widen the parameter type:

```ts
function OptionalEmail(input?: unknown): string | undefined {
  return input == null ? undefined : Email(input);
}
```

### Using `.transform()`

Wrap a custom validator with `.transform()` to enable chaining:

```ts
const EmailWithValidatorChain = unknown.string.transform(Email);

const UserSchema = Schema({
  email: EmailWithValidatorChain.optional().max(100),
});
```

### Asynchronous Validators

Return a `Promise` (or `PromiseLike`) from a validator to enable async validation:

```ts
async function AvailableUsername(input: string): Promise<string> {
  const res = await fetch(
    `/check-username?username=${encodeURIComponent(input)}`,
  );

  if (!res.ok) {
    throw new TypeError(`Username "${input}" is already taken`);
  }

  return input;
}

const UserSchema = Schema({
  username: AvailableUsername,
});

const user = await UserSchema({ username: 'test' });
```

`schema-checker` propagates async return types through the schema. Accessing a promise result without `await` is flagged by TypeScript.

## Validator Chains

Every built-in validator is a callable function with chainable helpers. Chaining runs validators in order and updates the inferred type as transforms are applied.

```ts
import { unknown } from 'schema-checker';

const validator = unknown.number().gt(0).toFixed(2);

console.log(validator('123.4567')); // '123.46'
```

After `.toFixed()`, the validator returns a `string`, so subsequent chain methods are string helpers.

### Common chain methods

| Method | Description |
| --- | --- |
| [`.equals()`](#equals) | Assert the value equals a given value |
| [`.test()`](#test) | Run a custom predicate |
| [`.transform()`](#transform) | Map the validated value to a new value |
| [`.construct()`](#construct) | Reshape arguments before validation |
| [`.optional()`](#optional) | Allow `null` or `undefined` |
| [`.strictOptional()`](#strictoptional) | Allow only `undefined` |
| [`.destruct()`](#destruct) | Return `[error, value]` instead of throwing |
| [`.error()`](#error) | Replace thrown errors with a custom message |

#### `.equals()`

```ts
const validator = boolean.equals(true);
```

#### `.test()`

```ts
import * as EmailValidator from 'email-validator';

const validator = string.test(EmailValidator.validate, 'Invalid email address');
```

#### `.transform()`

```ts
const validator = number.transform((x): number => {
  if (x <= 0) {
    throw new RangeError('Expected number to be positive');
  }

  return Math.sqrt(x);
});
```

#### `.construct()`

Reshape validator arguments before the underlying validator runs. The construct function must return an array of arguments.

```ts
const validator = number.gt(1).construct((x: number, y: number) => [x + y]);
validator(1, 2); // validates 3
```

#### `.optional()`

```ts
const validator = Schema({
  name: string.trim().min(1),
  address: string.trim().optional(),
});
```

#### `.strictOptional()`

Same as `.optional()`, but only `undefined` is accepted (not `null`).

```ts
const validator = Schema({
  name: string.trim().min(1),
  address: string.trim().strictOptional(),
});
```

#### `.destruct()`

Return a tuple `[error, value]` instead of throwing on validation failure.

```ts
const validator = Schema({
  name: string.trim().min(1),
}).destruct();

const [err, user] = validator(req.body);
```

#### `.error()`

Replace validation errors with a custom message, `ValidationError`, or error factory.

```ts
const validator = Schema({
  name: string.error('expect input to be string'),
  amount: number.gt(0, (val) => `${val} is not a positive amount`),
});
```

## API Reference

Import the primitives you need:

```ts
import Schema, {
  unknown,
  string,
  number,
  boolean,
  array,
  DateType,
} from 'schema-checker';
```

### `Schema`

Create a validator from a schema object, literal values, or function validators.

```ts
const validator = Schema(
  {
    name: string,
    amount: number,
  },
  'Missing name or amount',
);
```

**Strict mode** — Reject properties not defined on the schema:

```ts
const validator = Schema(
  {
    name: string,
    amount: number,
  },
  { strict: true },
);
```

**`Schema.either`** — Validate one of several shapes (OR):

```ts
const validator = Schema.either({ foo: string }, { bar: number });
// { foo: string } | { bar: number }
```

**`Schema.merge`** — Merge multiple schemas (AND):

```ts
const validator = Schema.merge({ foo: string }, { bar: number });
// { foo: string; bar: number }
```

**`Schema.enum`** — Validate against a TypeScript enum:

```ts
enum Status {
  OK,
  Invalid,
}

const validator = Schema.enum(Status, 'Invalid status');
```

**`Schema.record`** — Validate a `Record<key, value>`:

```ts
const validator = Schema.record(string.regexp(/^[a-z]+$/), number);
```

### `unknown`

Accept any value and coerce or validate it:

```ts
const validator = Schema({ data: unknown });
```

| Method | Description |
| --- | --- |
| `unknown.schema()` | Coerce to a nested schema |
| `unknown.object()` | Coerce to an object |
| `unknown.array()` | Coerce to an array |
| `unknown.string()` | Coerce to a string |
| `unknown.number()` | Coerce to a number |
| `unknown.boolean()` | Coerce to a boolean |
| `unknown.date()` | Coerce to a `Date` |
| `unknown.enum()` | Coerce to an enum value |
| `unknown.record()` | Coerce to a record |

```ts
const validator = unknown.string('Expect data to be string').toUpperCase();
// accepts { data: 1 } and converts to { data: '1' }
```

### `string`

Accept string values (including empty strings).

| Method | Description |
| --- | --- |
| `toLowerCase()` / `toUpperCase()` | Change case |
| `toLocaleLowerCase()` / `toLocaleUpperCase()` | Locale-aware case change |
| `trim()` | Remove leading and trailing whitespace |
| `truncate(n)` | Truncate with ellipsis |
| `normalize()` | Unicode normalization |
| `min(n)` / `max(n)` / `between(min, max)` | Length constraints |
| `regexp(pattern)` | Match a regular expression |

### `number`

Accept numeric values.

| Method | Description |
| --- | --- |
| `float()` | Accept floats (reject NaN and non-finite values) |
| `integer()` | Accept integers only |
| `toExponential()` / `toFixed()` / `toPrecision()` | Format as string |
| `toLocaleString()` | Locale-formatted string |
| `toString(radix?)` | Convert to string |
| `gte()` / `lte()` / `gt()` / `lt()` / `between()` | Range constraints |

### `boolean`

Accept boolean values.

```ts
const validator = Schema({ agree: boolean });
```

### `array`

Accept array values.

| Method | Description |
| --- | --- |
| `of(schema)` | Validate each element |
| `min(n)` / `max(n)` / `between(min, max)` | Length constraints |

```ts
const numbers = array.of(number);
const tuple = array.of(number).between(1, 2);
const objects = array.of({ foo: number });
const enums = array.of(Schema.enum(Status));
```

### `DateType`

Accept `Date` instances.

| Method | Description |
| --- | --- |
| `toISOString()` | Convert to ISO string |
| `getTime()` | Convert to timestamp |
| `gte()` / `lte()` / `gt()` / `lt()` / `between()` | Date range constraints |

```ts
const validator = Schema({ eventTime: DateType });
```

## Development

```bash
npm install    # install dependencies and build
npm test       # type-check, lint, and run tests
npm run build  # compile to lib/
```

## License

[MIT](LICENSE) &copy; 2020 [deepthought26](https://deepthought26.com)
