# DON (DataObjectNotation) Format Design

A binary-primary, JSON-compatible data format with first-class dataframe support.

## Overview

DON is a data serialization format designed for service-to-service API communication. It provides:

- Binary as primary representation (compact, fast)
- JSON as secondary representation (lossless round-trip, debugging)
- First-class columnar dataframes that can nest anywhere
- Rich type system beyond JSON primitives
- Schema-driven validation with constraints
- Deterministic serialization for content addressing

## Core Data Model

DON is a superset of JSON with additional types and structures.

### Primitive Types

| Category | Types |
|----------|-------|
| Null | `null` |
| Boolean | `bool` |
| Integers | `int8`, `int16`, `int32`, `int64`, `bigint` |
| Floats | `float32`, `float64`, `decimal` |
| Strings | `string` (UTF-8) |
| Binary | `bytes`, `uuid`, `fixed[N]` |
| Temporal | `date`, `time`, `datetime`, `duration` |
| Enums | named value sets with underlying type |

### Compound Types

| Type | Description |
|------|-------------|
| `object` | key-value map (string keys, ordered for determinism) |
| `array` | ordered sequence of values |
| `frame` | columnar table with typed columns |

### Frame Structure

A `frame` is a first-class columnar table that can appear anywhere a value can.

```
frame {
  columns: [Column...]
  row_count: uint64
}

column {
  name: string
  type: Type           # any DON type, including nested objects/arrays/frames
  nullable: bool
  values: [...]        # column-major storage
  null_bitmap?: bits   # if nullable, tracks which values are null
}
```

Columns can hold primitives or complex nested types (arrays, objects, other frames).

## Schema Definition

Schemas are defined natively in each target language.

### Python (pydantic-style)

```python
from don import DONFrame, Field, int64, datetime

class Status(DONEnum):
    ACTIVE = 1
    INACTIVE = 2
    PENDING = 3

class User(DONFrame):
    id: int64 = Field(unique=True)
    email: str = Field(pattern=r"^[\w]+@[\w]+\.[\w]+$", max_length=255)
    score: float64 = Field(ge=0, le=100)
    created: datetime
    tags: list[str] = Field(max_items=10)
    status: Status
```

### TypeScript

```typescript
import { frame, field, int64, datetime, enumType } from 'don';

const Status = enumType('Status', { ACTIVE: 1, INACTIVE: 2, PENDING: 3 });

const User = frame('User', {
  id: field(int64, { unique: true }),
  email: field.string({ pattern: /^[\w]+@[\w]+\.[\w]+$/, maxLength: 255 }),
  score: field.float64({ gte: 0, lte: 100 }),
  created: datetime,
  tags: field.array(field.string, { maxItems: 10 }),
  status: Status,
});
```

### Rust

```rust
use don::{DONFrame, field};

#[don_enum]
enum Status { Active = 1, Inactive = 2, Pending = 3 }

#[don_frame]
struct User {
    #[don(unique)]
    id: i64,
    #[don(pattern = r"^[\w]+@[\w]+\.[\w]+$", max_length = 255)]
    email: String,
    #[don(range = 0.0..=100.0)]
    score: f64,
    created: DateTime,
    #[don(max_items = 10)]
    tags: Vec<String>,
    status: Status,
}
```

## Constraints

Schemas express constraints enforced during serialization/deserialization.

| Constraint | Applies to | Example |
|------------|-----------|---------|
| `nullable` | any | `Field(nullable=True)` |
| `unique` | frame column | `Field(unique=True)` |
| `sorted` | frame column | `Field(sorted="asc")` or `"desc"` |
| `min`, `max` | numbers | `Field(ge=0, le=100)` |
| `min_length`, `max_length` | strings, bytes | `Field(max_length=255)` |
| `min_items`, `max_items` | arrays | `Field(max_items=10)` |
| `pattern` | strings | `Field(pattern=r"^\d{4}-\d{2}-\d{2}$")` |

## Validation

- **Default**: Strict - invalid data raises an error
- **Built-in coercion**: Optional, per-field (e.g., `"42"` → `42`)
- **Parsing callbacks**: User-provided hooks for custom lenient handling

```python
def my_parser(value, field_type, context):
    # custom lenient logic
    return coerced_value

result = User.parse(data, on_parse=my_parser)
```

## Binary Encoding

### Message Structure

```
DON Binary Message:
┌─────────────────────────────────────┐
│ Magic bytes (4): "DON\0"            │
│ Version (1): format version         │
│ Flags (1): compression, features    │
│ Root type tag (1)                   │
│ Payload (variable)                  │
└─────────────────────────────────────┘
```

### Encoding by Type

| Type | Encoding |
|------|----------|
| Integers | Varint (small values compact), fixed when schema specifies |
| Floats | IEEE 754 binary |
| Strings | Length-prefixed UTF-8 |
| Bytes | Length-prefixed raw |
| Temporal | int64 epoch + precision tag |
| Enums | Varint of underlying value |
| Arrays | Count + elements |
| Objects | Count + sorted key-value pairs |
| Frames | Column-major (see below) |

### Frame Encoding

```
Frame:
┌────────────────────────────────────┐
│ Row count (varint)                 │
│ Column count (varint)              │
│ For each column:                   │
│   ├─ Null bitmap (if nullable)     │
│   └─ Values (type-specific array)  │
└────────────────────────────────────┘
```

### Optimizations

- Null bitmap: 1 bit per row, packed
- Run-length encoding: for sorted/repetitive columns
- Dictionary encoding: for low-cardinality string columns (optional)

## JSON Representation

Schema defines types. JSON carries values as strings (parseable). Only `frame` needs a structural marker.

| DON Type | JSON Representation |
|----------|---------------------|
| `null`, `bool` | Native JSON |
| `int8`-`int64` | JSON number (if safe range) or string |
| `bigint` | String `"123456789012345678901234"` |
| `float32`, `float64` | JSON number |
| `decimal` | String `"123.456789"` |
| `bytes` | String (base64) |
| `uuid` | String `"550e8400-e29b-41d4-a716-446655440000"` |
| `date` | String `"2024-01-15"` |
| `datetime` | String `"2024-01-15T10:30:00.123456Z"` (always UTC, up to µs) |
| `duration` | String `"PT1H30M"` |
| `enum` | String `"ACTIVE"` or number `1` |
| `string` | Native JSON string |
| `frame` | `{"$frame": {...}}` |

### Frame in JSON

```json
{
  "$frame": {
    "columns": ["id", "name", "score"],
    "data": {
      "id": [1, 2, 3],
      "name": ["alice", "bob", "carol"],
      "score": [95.5, 87.2, 91.0]
    }
  }
}
```

## Schema Management

Schemas are agreed out-of-band and compiled into services. No registry.

```
                  schema source
                  (any language)
                       │
                       ▼
              ┌────────────────┐
              │  DON compiler  │
              └────────────────┘
                 │    │    │
         ┌───────┘    │    └───────┐
         ▼            ▼            ▼
      models.py   models.rs   models.ts
         │            │            │
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Service │  │ Service │  │  (TS)   │
    │ (Python)│  │ (Rust)  │  │ Service │
    └─────────┘  └─────────┘  └─────────┘
```

Binary messages have no schema ID. Both sides must have matching schemas.

## Developer API

### Python

```python
from don import DONFrame, encode, decode, to_json, from_json

class User(DONFrame):
    id: int64
    name: str
    created: datetime

# Create data
users = User.create([
    {"id": 1, "name": "alice", "created": datetime.now()},
    {"id": 2, "name": "bob", "created": datetime.now()},
])

# Binary encode/decode
binary = encode(users)
users = decode(binary, User)

# JSON encode/decode
json_str = to_json(users)
users = from_json(json_str, User)

# Access data
users["name"]           # column access → ["alice", "bob"]
users[0]                # row access → {"id": 1, "name": "alice", ...}
users["name"][0]        # cell access → "alice"
```

### Rust

```rust
let users = User::create(vec![...]);

let binary: Vec<u8> = don::encode(&users);
let users: User = don::decode(&binary)?;

let json: String = don::to_json(&users);
let users: User = don::from_json(&json)?;
```

### TypeScript

```typescript
const users = User.create([...]);

const binary: Uint8Array = don.encode(users);
const users = don.decode(binary, User);

const json: string = don.toJson(users);
const users = don.fromJson(json, User);
```

## Content Addressing

Deterministic serialization enables content addressing.

### Requirements for Determinism

- Object keys: sorted lexicographically (UTF-8 byte order)
- Frame columns: defined order from schema
- Frame rows: sorted by specified columns (if `sorted` constraint present)
- Numbers: canonical encoding (no leading zeros in varints)
- Strings: normalized UTF-8 (NFC normalization optional)

### Usage

```python
from don import encode, content_hash

users = User.create([...])
binary = encode(users, deterministic=True)
hash = content_hash(users)  # SHA-256 of deterministic binary

# Same data = same hash, regardless of insertion order
users1 = User.create([{"id": 1}, {"id": 2}])
users2 = User.create([{"id": 2}, {"id": 1}])

# If schema has: id: int64 = Field(sorted="asc")
content_hash(users1) == content_hash(users2)  # True
```

## Error Handling

```python
from don import ValidationError, SchemaError, DecodeError

# Validation error
try:
    users = User.parse({"id": "not-a-number", "name": "alice"})
except ValidationError as e:
    e.field      # "id"
    e.expected   # "int64"
    e.got        # "string: 'not-a-number'"
    e.path       # ["id"]
    e.message    # "Field 'id': expected int64, got string 'not-a-number'"

# Schema error
try:
    class Bad(DONFrame):
        x: int64 = Field(min=100, max=50)
except SchemaError as e:
    e.message    # "Field 'x': min (100) cannot exceed max (50)"

# Decode error
try:
    users = decode(corrupted_bytes, User)
except DecodeError as e:
    e.offset     # byte offset where error occurred
    e.message    # "Unexpected EOF at offset 42"

# Aggregate errors
result = User.parse(data, collect_errors=True)
if result.errors:
    for e in result.errors:
        print(e.path, e.message)
```

## Summary

| Aspect | Decision |
|--------|----------|
| **Primary format** | Binary (balanced size/speed) |
| **Secondary format** | JSON (lossless round-trip) |
| **Data model** | JSON superset + rich types + frames |
| **Types** | int8-64, bigint, float32/64, decimal, date, datetime, duration, bytes, uuid, enum, frame |
| **Frames** | Columnar tables, nestable anywhere, columns can hold complex types |
| **Schema location** | Out-of-band (compiled into services) |
| **Schema definition** | Native in Python, Rust, TypeScript |
| **Constraints** | nullability, ranges, patterns, sorted, unique |
| **Validation** | Strict default, optional coercion, parsing callbacks |
| **Temporal** | datetime (UTC, microsecond precision), date, duration |
| **Content addressing** | Deterministic serialization via sorted constraints |
| **Target languages** | Rust (core), Python, TypeScript |

## Out of Scope

- Schema registry
- Streaming/chunked encoding (could add later)
- Compression (delegated to transport layer)
- RPC framework (just a data format)
