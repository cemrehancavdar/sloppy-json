# Error Handling

sloppy-json provides detailed error information and partial recovery capabilities.

## Exceptions

### SloppyJSONError

Base exception class for all sloppy-json errors.

```python
from sloppy_json import SloppyJSONError

try:
    result = parse(text, options)
except SloppyJSONError as e:
    print(f"Parsing failed: {e}")
```

### SloppyJSONDecodeError

Raised when JSON decoding fails. Includes position information.

```python
from sloppy_json import parse_strict, SloppyJSONDecodeError

try:
    parse_strict('{invalid json}')
except SloppyJSONDecodeError as e:
    print(e.message)    # "Expected string for key"
    print(e.line)       # 1
    print(e.column)     # 2
    print(e.position)   # 1 (character offset from start)
```

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `message` | `str` | Error description |
| `line` | `int` | Line number (1-based) |
| `column` | `int` | Column number (1-based) |
| `position` | `int` | Character offset (0-based) |

#### Methods

```python
# Convert to ErrorInfo dataclass
error_info = e.to_error_info()
```

---

## ErrorInfo

Dataclass containing error details, used with partial recovery mode.

```python
from sloppy_json import ErrorInfo

@dataclass
class ErrorInfo:
    message: str
    position: int
    line: int
    column: int
```

String representation:

```python
error = ErrorInfo("Expected value", position=15, line=1, column=16)
print(str(error))  # "Expected value at line 1, column 16"
```

---

## Partial Recovery Mode

By default, parsing stops at the first error. With `partial_recovery=True`, parsing continues and collects all errors.

### Basic Usage

```python
from sloppy_json import parse, RecoveryOptions

opts = RecoveryOptions(partial_recovery=True)

result, errors = parse('{"a": 1, "b": , "c": 3}', opts)

print(result)  # '{"a": 1, "b": null, "c": 3}'
print(len(errors))  # 1

for error in errors:
    print(f"Line {error.line}: {error.message}")
```

### Return Type

When `partial_recovery=True`:
- Returns: `tuple[str, list[ErrorInfo]]`

When `partial_recovery=False` (default):
- Returns: `str`
- Raises: `SloppyJSONDecodeError` on error

### Recovery Behavior

In partial recovery mode:
- Missing values become `null`
- Invalid tokens are skipped where possible
- Parsing continues to find more errors

```python
opts = RecoveryOptions(partial_recovery=True)

# Multiple errors
text = '{"a": , "b": undefined, "c": }'
result, errors = parse(text, opts)

print(result)  # '{"a": null, "b": null, "c": null}'
print(len(errors))  # 3 errors collected
```

### Error Reporting

```python
opts = RecoveryOptions(partial_recovery=True)
result, errors = parse(broken_json, opts)

if errors:
    print(f"Parsed with {len(errors)} error(s):")
    for error in errors:
        print(f"  - {error}")
else:
    print("Parsed successfully")

print(f"Result: {result}")
```

---

## Common Error Messages

| Message | Cause |
|---------|-------|
| `Expected string for key` | Object key is not a quoted string |
| `Expected ':'` | Missing colon after object key |
| `Expected ',' or '}'` | Missing comma or closing brace |
| `Expected value` | Missing value after colon or in array |
| `Unexpected token` | Invalid character in context |
| `Unterminated string` | String not closed before end of input |
| `Invalid escape sequence` | Bad `\x` in string |
| `Trailing data after JSON` | Extra content after valid JSON |

---

## Best Practices

### 1. Use Appropriate Mode

```python
# For validation: strict mode, catch errors
try:
    result = parse_strict(text)
except SloppyJSONDecodeError as e:
    log_error(f"Invalid JSON at line {e.line}: {e.message}")

# For recovery: permissive mode
result = parse_permissive(text)

# For debugging: partial recovery
result, errors = parse(text, RecoveryOptions(partial_recovery=True))
if errors:
    for e in errors:
        log_warning(f"Recovered from: {e}")
```

### 2. Log Errors for Monitoring

```python
def parse_with_logging(text):
    opts = RecoveryOptions.permissive()
    opts = replace(opts, partial_recovery=True)
    
    result, errors = parse(text, opts)
    
    if errors:
        logger.warning(f"JSON recovery: {len(errors)} issues")
        for error in errors:
            logger.debug(f"  {error}")
    
    return result
```

### 3. Validate After Recovery

```python
import json

def safe_parse(text):
    result = parse_permissive(text)
    
    # Verify it's valid JSON
    try:
        json.loads(result)
        return result
    except json.JSONDecodeError:
        raise ValueError("Recovery produced invalid JSON")
```

---

## Type Hints

For type checkers, the return type depends on `partial_recovery`:

```python
from sloppy_json import parse, RecoveryOptions, ErrorInfo

# Without partial_recovery
def process(text: str) -> str:
    return parse(text, RecoveryOptions())

# With partial_recovery  
def process_safe(text: str) -> tuple[str, list[ErrorInfo]]:
    opts = RecoveryOptions(partial_recovery=True)
    return parse(text, opts)
```
