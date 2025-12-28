<p align="center">
  <img src="assets/logo.png" alt="sloppy-json" width="400">
</p>

# sloppy-json

A forgiving JSON parser that recovers broken JSON from LLM outputs.

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

## Usage

```python
from sloppy_json import parse, parse_lenient, parse_permissive, RecoveryOptions

# Strict parsing (standard JSON only)
result = parse('{"key": "value"}')

# Lenient parsing (common LLM issues)
result = parse_lenient("{'key': 'value',}")  # single quotes + trailing comma

# Permissive parsing (maximum recovery)
result = parse_permissive("Here is the JSON: {name: 'test'")  # everything

# Custom options
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

## Auto-detection

Automatically detect what options are needed:

```python
from sloppy_json import detect_required_options

samples = ["{'key': 'value',}", "{name: True}"]
options = detect_required_options(samples)
# options.allow_single_quotes == True
# options.allow_trailing_commas == True
# options.allow_unquoted_keys == True
# options.convert_python_literals == True
```

## Documentation

- [Getting Started](docs/index.md) - Overview and quick start
- [Options Reference](docs/options.md) - All `RecoveryOptions` explained
- [Presets Guide](docs/presets.md) - When to use strict/lenient/permissive
- [Auto-Detection](docs/detection.md) - Automatically detect required options
- [Error Handling](docs/errors.md) - Exceptions and partial recovery
- [Examples](docs/examples.md) - Common scenarios and recipes

## License

MIT
