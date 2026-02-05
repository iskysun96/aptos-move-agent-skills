# Error Codes Reference

## Aptos Framework Error Codes

Common framework error codes:

| Code  | Error             | Meaning                     |
| ----- | ----------------- | --------------------------- |
| `0x1` | INVALID_ARGUMENT  | Invalid function argument   |
| `0x2` | OUT_OF_RANGE      | Value out of valid range    |
| `0x3` | INVALID_STATE     | Invalid state for operation |
| `0x5` | NOT_FOUND         | Resource not found          |
| `0x6` | ALREADY_EXISTS    | Resource already exists     |
| `0x7` | PERMISSION_DENIED | Caller not authorized       |
| `0x8` | ABORTED           | Operation aborted           |

## Custom Error Code Best Practices

Organize error codes by category:

```move
// Access control: 1-9
const E_NOT_OWNER: u64 = 1;
const E_NOT_ADMIN: u64 = 2;
const E_UNAUTHORIZED: u64 = 3;

// Input validation: 10-19
const E_ZERO_AMOUNT: u64 = 10;
const E_AMOUNT_TOO_HIGH: u64 = 11;
const E_INVALID_ADDRESS: u64 = 12;

// State errors: 20-29
const E_NOT_INITIALIZED: u64 = 20;
const E_ALREADY_INITIALIZED: u64 = 21;
const E_PAUSED: u64 = 22;

// Business logic: 30+
const E_INSUFFICIENT_BALANCE: u64 = 30;
const E_ITEM_NOT_AVAILABLE: u64 = 31;
```

## Debugging Error Codes

When you see `ABORTED: 0x1E` (hex 0x1E = decimal 30):

1. Convert hex to decimal: 0x1E = 30
2. Find constant with value 30
3. Identify which assertion failed
4. Fix the underlying issue

## See Also

- Main SKILL.md for error handling patterns
- `error-catalog.md` for complete error database
