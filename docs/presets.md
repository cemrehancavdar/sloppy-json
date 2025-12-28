# Presets Guide

sloppy-json provides three presets for common use cases. Choose the right one based on your needs.

## Quick Comparison

| Preset | Use Case | Recovery Level |
|--------|----------|----------------|
| `strict()` | Standard JSON only | None |
| `lenient()` | Common LLM issues | Moderate |
| `permissive()` | Maximum recovery | Aggressive |

## Strict Mode

**Use when:** You expect valid JSON and want to catch errors.

```python
from sloppy_json import parse_strict, RecoveryOptions

# These work
parse_strict('{"key": "value"}')
parse_strict('[1, 2, 3]')
parse_strict('null')

# These raise SloppyJSONDecodeError
parse_strict("{'key': 'value'}")     # single quotes
parse_strict('{"a": 1,}')            # trailing comma
parse_strict('{key: "value"}')       # unquoted key
```

### Options Enabled

None. Equivalent to `RecoveryOptions()` with all defaults.

### When to Use

- Validating JSON from trusted sources
- Testing that your code produces valid JSON
- When you need strict compliance

---

## Lenient Mode

**Use when:** Parsing LLM output that's mostly correct but has common issues.

```python
from sloppy_json import parse_lenient

# Single quotes
parse_lenient("{'name': 'John'}")
# Returns: '{"name": "John"}'

# Trailing commas
parse_lenient('{"items": [1, 2, 3,],}')
# Returns: '{"items": [1, 2, 3]}'

# Python literals
parse_lenient('{"active": True, "data": None}')
# Returns: '{"active": true, "data": null}'

# Code blocks
parse_lenient('```json\n{"key": "value"}\n```')
# Returns: '{"key": "value"}'

# Comments
parse_lenient('{"a": 1} // this is a comment')
# Returns: '{"a": 1}'
```

### Options Enabled

```python
RecoveryOptions(
    allow_single_quotes=True,
    allow_trailing_commas=True,
    convert_python_literals=True,
    extract_from_code_blocks=True,
    extract_from_triple_quotes=True,
    allow_comments=True,
)
```

### When to Use

- Parsing ChatGPT/Claude/LLM responses
- JSON that might have Python-style values
- JSON wrapped in markdown code blocks
- When you want recovery but not too aggressive

### What It Doesn't Handle

```python
# These still raise errors in lenient mode
parse_lenient('{key: "value"}')       # unquoted keys
parse_lenient('{"a": 1 "b": 2}')      # missing commas
parse_lenient('{"key": "value"')      # truncated JSON
parse_lenient('Here is: {"a": 1}')    # text extraction
```

---

## Permissive Mode

**Use when:** You need maximum recovery and don't care about strictness.

```python
from sloppy_json import parse_permissive

# Everything lenient handles, plus...

# Unquoted keys
parse_permissive('{name: "John", age: 30}')
# Returns: '{"name": "John", "age": 30}'

# Missing commas
parse_permissive('{"a": 1 "b": 2 "c": 3}')
# Returns: '{"a": 1, "b": 2, "c": 3}'

# Truncated JSON
parse_permissive('{"users": [{"name": "Alice"}, {"name": "Bob"')
# Returns: '{"users": [{"name": "Alice"}, {"name": "Bob"}]}'

# Text extraction
parse_permissive('The result is {"status": "ok"} as expected.')
# Returns: '{"status": "ok"}'

# Unquoted identifiers
parse_permissive('{"status": success}')
# Returns: '{"status": "success"}'

# Unescaped newlines
parse_permissive('{"text": "line1\nline2"}')
# Returns: '{"text": "line1\\nline2"}'

# Complex example
text = '''Here's your data:
```json
{
  name: 'test',  // a comment
  active: True,
  items: [1, 2, 3,],
'''
parse_permissive(text)
# Returns: '{"name": "test", "active": true, "items": [1, 2, 3]}'
```

### Options Enabled

```python
RecoveryOptions(
    allow_unquoted_keys=True,
    allow_single_quotes=True,
    allow_unquoted_identifiers=True,
    allow_trailing_commas=True,
    allow_missing_commas=True,
    auto_close_objects=True,
    auto_close_arrays=True,
    auto_close_strings=True,
    extract_json_from_text=True,
    extract_from_code_blocks=True,
    extract_from_triple_quotes=True,
    convert_python_literals=True,
    handle_undefined="null",
    handle_nan="string",
    handle_infinity="string",
    allow_comments=True,
    escape_newlines_in_strings=True,
)
```

### When to Use

- Parsing unreliable LLM output
- Recovering from truncated responses (token limits)
- When you absolutely need to get JSON out
- Scraping or processing messy data

### Trade-offs

Permissive mode may occasionally interpret non-JSON text as JSON. For example:

```python
parse_permissive('The values are x=1 and y=2')
# Might extract something unexpected
```

If you need more control, use custom options instead.

---

## Custom Options

When presets don't fit, build your own:

```python
from sloppy_json import parse, RecoveryOptions

# Like lenient, but also handle truncated JSON
opts = RecoveryOptions(
    allow_single_quotes=True,
    allow_trailing_commas=True,
    convert_python_literals=True,
    extract_from_code_blocks=True,
    auto_close_objects=True,
    auto_close_arrays=True,
    auto_close_strings=True,
)

result = parse(broken_json, opts)
```

See [Options Reference](options.md) for all available options.

---

## Choosing the Right Preset

```
Is the JSON from a trusted source?
├─ Yes → use strict()
└─ No → Is it from an LLM?
         ├─ Yes → Is the output often truncated?
         │        ├─ Yes → use permissive()
         │        └─ No → use lenient()
         └─ No → use permissive() or custom options
```
