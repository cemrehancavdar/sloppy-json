# sloppy-json

A forgiving JSON parser that recovers broken JSON from LLM outputs.

## Why sloppy-json?

LLMs often produce JSON that isn't quite valid:
- Single quotes instead of double quotes
- Trailing commas
- Unquoted keys
- Python literals (`True`, `False`, `None`)
- Truncated output
- Markdown code blocks around JSON
- JavaScript values (`undefined`, `NaN`, `Infinity`)

Standard `json.loads()` fails on all of these. sloppy-json recovers them.

## Installation

```bash
uv add sloppy-json
```

or with pip:

```bash
pip install sloppy-json
```

## Quick Start

```python
from sloppy_json import parse_permissive

# Handles almost any broken JSON
result = parse_permissive("{'name': 'test', 'active': True,}")
# Returns: '{"name": "test", "active": true}'
```

## Basic Usage

### Convenience Functions

```python
from sloppy_json import parse_strict, parse_lenient, parse_permissive

# Strict: standard JSON only (like json.loads)
parse_strict('{"key": "value"}')

# Lenient: common LLM issues (quotes, trailing commas, Python literals)
parse_lenient("{'key': True,}")

# Permissive: maximum recovery (all options enabled)
parse_permissive("{key: 'value'")  # truncated, unquoted key, single quotes
```

### Custom Options

```python
from sloppy_json import parse, RecoveryOptions

opts = RecoveryOptions(
    allow_single_quotes=True,
    allow_trailing_commas=True,
    convert_python_literals=True,
)
result = parse("{'flag': True,}", opts)
```

## Features

| Feature | Example Input | Output |
|---------|---------------|--------|
| Single quotes | `{'key': 'value'}` | `{"key": "value"}` |
| Unquoted keys | `{key: "value"}` | `{"key": "value"}` |
| Trailing commas | `{"a": 1,}` | `{"a": 1}` |
| Missing commas | `{"a": 1 "b": 2}` | `{"a": 1, "b": 2}` |
| Python literals | `{"flag": True}` | `{"flag": true}` |
| Truncated JSON | `{"key": "val` | `{"key": "val"}` |
| Code blocks | `` ```json {"a":1}``` `` | `{"a": 1}` |
| Comments | `{"a": 1} // comment` | `{"a": 1}` |
| JS undefined | `{"a": undefined}` | `{"a": null}` |
| NaN/Infinity | `{"a": NaN}` | `{"a": "NaN"}` |

## API Reference

### Functions

#### `parse(text: str, options: RecoveryOptions = None) -> str`

Parse JSON with specified recovery options. Returns normalized JSON string.

#### `parse_strict(text: str) -> str`

Parse standard JSON only. Equivalent to `parse(text, RecoveryOptions.strict())`.

#### `parse_lenient(text: str) -> str`

Parse with common LLM issue recovery. Equivalent to `parse(text, RecoveryOptions.lenient())`.

#### `parse_permissive(text: str) -> str`

Parse with maximum recovery. Equivalent to `parse(text, RecoveryOptions.permissive())`.

#### `detect_required_options(samples: list[str]) -> RecoveryOptions`

Analyze sample JSON strings and return minimum required options to parse them.

### Classes

#### `RecoveryOptions`

Configuration dataclass for recovery behavior. See [Options Reference](options.md).

#### `SloppyJSONDecodeError`

Exception raised when parsing fails. Includes position information.

```python
from sloppy_json import parse_strict, SloppyJSONDecodeError

try:
    parse_strict('{invalid}')
except SloppyJSONDecodeError as e:
    print(e.message)   # "Expected string for key"
    print(e.line)      # 1
    print(e.column)    # 2
    print(e.position)  # 1
```

#### `ErrorInfo`

Dataclass containing error details, used with `partial_recovery` mode.

## Documentation

- [Options Reference](options.md) - All `RecoveryOptions` explained
- [Presets Guide](presets.md) - When to use strict/lenient/permissive
- [Auto-Detection](detection.md) - Automatically detect required options
- [Error Handling](errors.md) - Exceptions and partial recovery
- [Examples](examples.md) - Common scenarios and recipes
