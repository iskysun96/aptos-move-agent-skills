# Move to TypeScript Type Mappings

## Primitive Types

| Move Type | TypeScript Type    | Notes                                                           |
| --------- | ------------------ | --------------------------------------------------------------- |
| `u8`      | `number`           | 0 to 255                                                        |
| `u16`     | `number`           | 0 to 65535                                                      |
| `u32`     | `number`           | 0 to 4294967295                                                 |
| `u64`     | `number \| bigint` | Safe as `number` up to 2^53 - 1; use `bigint` for larger values |
| `u128`    | `bigint`           | ALWAYS use `bigint` - `number` loses precision                  |
| `u256`    | `bigint`           | ALWAYS use `bigint` - `number` loses precision                  |
| `i8`      | `number`           | -128 to 127 (Move 2.3+)                                        |
| `i16`     | `number`           | -32,768 to 32,767 (Move 2.3+)                                  |
| `i32`     | `number`           | -2^31 to 2^31-1 (Move 2.3+)                                    |
| `i64`     | `number \| bigint` | Use `bigint` for values outside safe integer range (Move 2.3+)  |
| `i128`    | `bigint`           | ALWAYS use `bigint` (Move 2.3+)                                 |
| `i256`    | `bigint`           | ALWAYS use `bigint` (Move 2.3+)                                 |
| `bool`    | `boolean`          | `true` or `false`                                               |
| `address` | `string`           | Hex string, e.g., `"0x1"`                                       |

## String and Bytes

| Move Type    | TypeScript Type        | Notes                           |
| ------------ | ---------------------- | ------------------------------- |
| `String`     | `string`               | UTF-8 string                    |
| `vector<u8>` | `Uint8Array \| string` | Raw bytes or hex-encoded string |

## Collections

| Move Type   | TypeScript Type | Notes                   |
| ----------- | --------------- | ----------------------- |
| `vector<T>` | `T[]`           | Array of the inner type |
| `Option<T>` | `T \| null`     | Value present or null   |

## Object Types

| Move Type   | TypeScript Type | Notes                                          |
| ----------- | --------------- | ---------------------------------------------- |
| `Object<T>` | `string`        | Pass the object's address as a hex string      |
| `&signer`   | N/A             | Automatically filled by the transaction signer |

---

## Function Argument Examples

```typescript
// String
functionArguments: ["Hello World"];

// u8, u16, u32
functionArguments: [42];

// u64 (small values - safe as number)
functionArguments: [1000000];

// u128, u256 (ALWAYS use bigint or string)
functionArguments: [BigInt("340282366920938463463374607431768211455")];

// address
functionArguments: ["0x1abc...def"];

// Object<T> - pass the object's address
functionArguments: ["0x1234...object_address"];

// bool
functionArguments: [true];

// vector<u8> (bytes)
functionArguments: [new Uint8Array([1, 2, 3, 4])];

// vector<u64>
functionArguments: [[1, 2, 3, 4, 5]];

// vector<address>
functionArguments: [["0x1", "0x2", "0x3"]];
```

---

## Common Pitfalls

| Pitfall                            | Problem                          | Solution                           |
| ---------------------------------- | -------------------------------- | ---------------------------------- |
| Using `number` for u128            | Precision loss above 2^53        | Use `bigint`                       |
| Passing `number` for large u64     | Precision loss for values > 2^53 | Use `bigint` or string             |
| Forgetting to convert view results | Raw string from API              | Cast with `Number()` or `BigInt()` |
| Passing `Object<T>` as object      | SDK expects address string       | Pass `objectAddress.toString()`    |
| Not wrapping vector args in array  | SDK interprets as multiple args  | Wrap in `[...]`: `[[1,2,3]]`       |
