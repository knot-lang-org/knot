# Knot

**A modern configuration language with native references, imports, and zero indentation requirements.**

Knot combines the readability of YAML, the organization of TOML, and the structured data types of JSON into a lightweight configuration format.

Unlike traditional configuration languages, Knot treats references as first-class values, allowing configuration properties, objects, and entire sections to be shared across files without duplication.

## Features

- **Indentation-independent** — Whitespace is for readability, not structure.
- **Section-based configuration** — Organize properties using `[section]` and `[section.subsection]`.
- **Native JSON values** — Use arrays, objects, booleans, numbers, and `null`. Any valid JSON value is a valid Knot value.
- **Native dates and times** — RFC 3339 date-times, dates, and times as typed values.
- **Unquoted strings** — No unnecessary quotation marks.
- **Configuration pointers** — Reference properties and entire sections using `@`.
- **Imports** — Import other `.knot` files and reference their sections and properties.
- **Typed references** — Preserve the original data type when resolving pointers.
- **Configuration inheritance** — Extend existing sections and override individual properties.
- **Lazy resolution, eager validation** — Applications load only what they read; `knot validate` checks everything.
- **Circular-reference detection** — Identify invalid dependency chains.
- **Strict validation** — Detect duplicate keys, conflicting types, and malformed references.

## Quick Start

Create a `database.knot` file:

```ini
[production]
driver: mysql
host: production-db.internal
port: 3306
```

Create an `app.knot` file next to it:

```ini
# Application configuration
@import db: "./database.knot"

[app]
name: My Application
environment: production
debug: false
workers: 8

[database] < @db.production
username: app
password: @env.DB_PASSWORD

[cache]
driver: redis
host: @database.host
port: 6379
enabled: @app.debug

[queue]
workers: @app.workers
```

### Resolved Configuration

Knot resolves references while preserving their original types. With environment access enabled and `DB_PASSWORD=s3cret`, `app.knot` resolves to:

```json
{
    "app": {
        "name": "My Application",
        "environment": "production",
        "debug": false,
        "workers": 8
    },
    "database": {
        "driver": "mysql",
        "host": "production-db.internal",
        "port": 3306,
        "username": "app",
        "password": "s3cret"
    },
    "cache": {
        "driver": "redis",
        "host": "production-db.internal",
        "port": 6379,
        "enabled": false
    },
    "queue": {
        "workers": 8
    }
}
```

## Syntax

### File Structure

A Knot file has up to three parts, in this order:

1. Imports (`@import`)
2. Root properties, which belong to the top-level object
3. Sections

```ini
@import db: "./database.knot"

version: 2
name: My Application

[server]
port: 8080
```

Files are UTF-8 without a byte order mark. Lines may end in LF or CRLF.

### Comments

`#` starts a comment when it is the first non-whitespace character on a line, or when it has whitespace on both sides.

```ini
# Full-line comment
[theme]
debug: true # inline comment
color: #ff6600
docs: https://example.com/guide#install
```

`color` is `"#ff6600"` and `docs` is `"https://example.com/guide#install"`, because neither `#` has whitespace on both sides. A `#` inside a quoted string is never a comment.

### Sections

Sections are declared using square brackets.

```ini
[database]
host: localhost
port: 3306

[database.credentials]
username: root
password: secret
```

Nested section names use dot notation.

Each section header may appear only once per file. Subsections can be declared anywhere in the file, including after unrelated sections:

```ini
[database]
host: localhost

[cache]
driver: redis

[database.replica]
host: replica.internal
```

Duplicate property assignments within the same configuration scope are errors.

### Keys

Bare keys may contain ASCII letters, digits, `_`, and `-`.

Keys containing other characters, including dots, must be quoted:

```ini
[hosts]
"api.example.com": 443

[hosts."cdn.example.com"]
port: 443
```

### Data Types

Knot supports JSON-compatible primitive and structured values.

| Type | Example |
|---|---|
| String | `name: My Application` |
| Quoted string | `name: "My Application"` |
| Multiline string | `motd: """ ... """` |
| Integer | `port: 3306` |
| Float | `timeout: 1.5` |
| Boolean | `enabled: true` |
| Null | `password: null` |
| Offset date-time | `published: 2026-11-06T09:00:00Z` |
| Local date-time | `starts: 2026-11-06T09:00:00` |
| Local date | `released: 2026-11-06` |
| Local time | `backup_at: 02:30:00` |
| Array | `ports: [80, 443]` |
| Object | `options: {"timeout": 30}` |
| Reference | `host: @database.host` |
| Environment variable | `password: @env.DB_PASSWORD` |

### Unquoted Strings

An unquoted value starts after the first `:` on the line and runs to the end of the line or to a comment. Leading and trailing whitespace is trimmed. Values may contain colons, so `url: http://localhost:8080` is the string `"http://localhost:8080"`.

Knot decides the type of an unquoted value from its first character:

| Starts with | Parsed as | If it doesn't parse |
|---|---|---|
| A digit | Number, date, or time | Error |
| `-`, `+`, or `.` | Number | Error |
| `@` | Reference | Error |
| `[` or `{` | Inline array or object | Error |
| `"` | Quoted or multiline string | Error |
| Anything else | `true`, `false`, `null`, or a string | — |

So `version: 1.2.3` and `zip: 01234` are errors, not strings. Quote them: `version: "1.2.3"`.

`version: 1.10` is valid, but it is the number `1.1`. Quote version numbers.

`true`, `false`, and `null` are lowercase. Other casings such as `True` or `NULL` are errors, so they can't silently become strings.

An empty value (`key:`) is an error. Use `null` or `""`.

### Quoted Strings

Quoted strings use JSON string syntax and escapes: `\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, and `\uXXXX`.

### Multiline Strings

Multiline strings use triple quotes.

```ini
[server]
motd: """
Welcome to the server.
Maintenance window: Sundays 02:00 UTC.
"""
```

A newline directly after the opening `"""` is dropped. Everything else, including indentation, is kept as written. Escapes work as in quoted strings.

### Numbers

Numbers use JSON number syntax, with `_` allowed between digits for readability:

```ini
[limits]
max_upload: 10_485_760
ratio: 0.75
offset: -3
scale: 1.5e3
```

Hex, octal, `Infinity`, and `NaN` are not supported.

### Dates and Times

Knot has four date and time types, written unquoted in [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) format. RFC 3339 is a strict profile of ISO 8601, so every Knot date is also a valid ISO 8601 date. ISO 8601 forms outside RFC 3339, such as `20261106` or `2026-W45`, are not supported because they collide with numbers.

```ini
[release]
published: 2026-11-06T09:00:00Z         # offset date-time
announced: 2026-11-06T09:00:00-05:00    # offset date-time
starts: 2026-11-06T09:00:00             # local date-time
released: 2026-11-06                    # local date
backup_at: 02:30                        # local time
```

| Type | Format | Meaning |
|---|---|---|
| Offset date-time | `YYYY-MM-DDTHH:MM[:SS[.fff]]Z` or `±HH:MM` | An exact instant |
| Local date-time | `YYYY-MM-DDTHH:MM[:SS[.fff]]` | A date and time with no time zone; the application decides which zone applies |
| Local date | `YYYY-MM-DD` | A calendar date |
| Local time | `HH:MM[:SS[.fff]]` | A time of day |

Rules:

- The date and time are separated by an uppercase `T`. The UTC designator is an uppercase `Z`.
- Seconds are optional and default to `00`.
- Fractional seconds are allowed. Implementations must keep at least millisecond precision and truncate anything beyond what they support.
- Values must be real dates and times. `2026-02-30` and `25:00` are errors.
- Quoted values are strings. `"2026-11-06"` stays a string.
- Dates and times are allowed unquoted inside inline objects and arrays: `holidays: [2026-01-01, 2026-07-04]`.
- References preserve date and time types like any other type.

### Dates in JSON Output

JSON has no date type. When resolved configuration is serialized to JSON, dates and times become RFC 3339 strings, normalized to include seconds:

```json
{
    "release": {
        "published": "2026-11-06T09:00:00Z",
        "released": "2026-11-06",
        "backup_at": "02:30:00"
    }
}
```

Library APIs return native date and time values. Only JSON serialization converts them to strings.

### Inline Objects and Arrays

Objects and arrays can be written inline using JSON syntax.

```ini
[database]
options: {
    "charset": "utf8mb4",
    "persistent": true,
    "timeout": 30,
    "retry": {
        "attempts": 3,
        "delay": 1000
    }
}
```

Any valid JSON value is a valid inline value. Knot adds four things to JSON inside inline values:

- References as values
- Unquoted dates and times
- `#` comments
- Trailing commas

```ini
[server]
hosts: [
    @db.production.host,
    "backup.internal",   # failover
]
```

Inside inline values, keys and strings must be quoted. An inline value can span multiple lines and ends when its brackets close.

### Arrays

```ini
[server]
ports: [80, 443, 8080]
hosts: ["web1.local", "web2.local"]
```

Arrays can contain nested objects.

```ini
[server]
nodes: [
    {"name": "web1", "host": "192.168.1.10"},
    {"name": "web2", "host": "192.168.1.11"}
]
```

### Arrays of Sections

Repeated array-section headers create ordered collections of objects.

```ini
[[servers]]
name: web1
host: 192.168.1.10

[[servers]]
name: web2
host: 192.168.1.11
```

Equivalent JSON:

```json
{
    "servers": [
        {
            "name": "web1",
            "host": "192.168.1.10"
        },
        {
            "name": "web2",
            "host": "192.168.1.11"
        }
    ]
}
```

### Indentation

Indentation has no structural significance.

Both examples are equivalent:

```ini
[database]
    host: localhost
    port: 3306
```

```ini
[database]
host: localhost
port: 3306
```

Configuration hierarchy is determined by section headers and inline objects and arrays.

---

# Configuration References

References are a core feature of Knot.

The `@` operator identifies another configuration value.

References preserve the original value's type rather than converting it into a string.

## Property References

```ini
[app]
debug: false
workers: 8

[queue]
debug: @app.debug
workers: @app.workers
```

The resolved values are:

```json
{
    "queue": {
        "debug": false,
        "workers": 8
    }
}
```

## Section References

References can target entire configuration sections.

```ini
[database.primary]
driver: mysql
host: db.internal
port: 3306

[database.reporting]
connection: @database.primary
```

The `connection` property resolves to an object containing the properties of `database.primary`.

## Reference Paths

A reference path is a sequence of keys separated by dots. Array elements are selected with a zero-based index in brackets, and keys that need quoting are quoted:

```ini
first_host: @servers[0].host
api_port: @hosts."api.example.com"
```

## Chained References

A reference can pass through another reference. If `database.connection` is a reference to a section, `@database.connection.host` follows it and reads `host` from that section.

```ini
[database.primary]
host: db.internal

[database]
connection: @database.primary

[cache]
host: @database.connection.host
```

`cache.host` resolves to `"db.internal"`.

## Imports

A Knot file can import other Knot files and reference their contents.

```ini
@import db: "./database.knot"
```

The name before the colon (`db`) becomes a reference root. `@db.production` refers to the `production` section of `database.knot`, and `@db.production.host` refers to a single property in it.

Consider the following project structure:

```text
config/
├── app.knot
└── database.knot
```

**database.knot**

```ini
[production]
driver: mysql
host: production-db.internal
port: 3306

[development]
driver: mysql
host: localhost
port: 3306
```

**app.knot**

```ini
@import db: "./database.knot"

[app]
name: My Application
environment: production
debug: false

[database]
connection: @db.production

[cache]
host: @database.connection.host
port: 6379

[database.replica] < @database.connection
host: replica.internal
```

Resolved configuration:

```json
{
    "app": {
        "name": "My Application",
        "environment": "production",
        "debug": false
    },
    "database": {
        "connection": {
            "driver": "mysql",
            "host": "production-db.internal",
            "port": 3306
        },
        "replica": {
            "driver": "mysql",
            "host": "replica.internal",
            "port": 3306
        }
    },
    "cache": {
        "host": "production-db.internal",
        "port": 6379
    }
}
```

### Import Rules

- Only `.knot` files can be imported. The path must include the `.knot` extension.
- Imports must appear before root properties and the first section header.
- Relative paths are resolved from the directory containing the importing file.
- Each import name must be unique within a file and cannot share a name with a root property or top-level section in that file.
- `env` is reserved and cannot be used as an import name.
- References resolve against the file they are written in. If `database.knot` contains `@defaults.host`, it resolves against the sections of `database.knot`, not the file that imported it.
- Imports are not transitive. If `database.knot` imports `shared.knot`, `app.knot` must import `shared.knot` itself to reference it.
- Two files may import each other. Only circular reference chains are errors.

## Environment Variables

`@env.NAME` reads the environment variable `NAME`.

```ini
[database]
host: @env.DB_HOST
password: @env.DB_PASSWORD
```

- Environment access is disabled by default. The application or `knot validate` must enable it. Referencing `@env` while it is disabled is an error.
- Values are always strings. `@env.PORT` resolves to `"8080"`, not `8080`.
- An unset variable is a reference error.
- `env` is a reserved root. A file cannot declare a root property, top-level section, or import named `env`.

---

# Configuration Inheritance

Sections can inherit properties from any reference that resolves to an object: another section, a property holding an object, or a section in an imported file.

The `<` operator declares inheritance.

```ini
[database.defaults]
driver: mysql
port: 3306
charset: utf8mb4
timeout: 30

[database.production] < @database.defaults
host: production-db.internal
timeout: 60

[database.development] < @database.defaults
host: localhost
```

The resolved production configuration becomes:

```json
{
    "driver": "mysql",
    "port": 3306,
    "charset": "utf8mb4",
    "timeout": 60,
    "host": "production-db.internal"
}
```

Explicit properties override inherited properties.

The parent can come from an imported file or from a property that references a section:

```ini
@import db: "./database.knot"

[database]
connection: @db.production

[database.replica] < @database.connection
host: replica.internal
```

`database.replica` gets `driver` and `port` from the `production` section of `database.knot` and overrides `host`.

## Inheritance Rules

- A section has at most one parent.
- A parent may itself inherit from another section. The parent's chain is resolved first.
- Inheritance cycles are errors, reported the same way as circular references.
- Inheritance is shallow. A parent's subsections are inherited as whole objects, like any other property. If the child defines a subsection or property with the same name, it replaces the inherited one entirely rather than merging with it.
- Inherited properties cannot be removed. A child can set one to `null`.
- Array sections can inherit: `[[servers]] < @defaults.server`.
- Inheriting from a reference that resolves to a scalar or an array is an error.

---

# Reference Resolution

Knot separates parsing from reference resolution.

## Lazy Resolution

During parsing, references are represented as configuration nodes rather than immediately evaluated.

For example:

```ini
@import db: "./database.knot"

[database]
host: @db.production.host
```

The parser records the import and the reference without immediately loading `database.knot`.

When the application requests `database.host`, the resolver:

1. Identifies the reference.
2. Loads the target configuration if necessary.
3. Locates the referenced node.
4. Resolves any nested references.
5. Returns the resulting typed value.
6. Caches the result for subsequent access.

Applications skip loading configuration dependencies they never read.

## Eager Validation

Lazy resolution means a broken reference in a value nobody reads produces no error at runtime. To catch those, Knot defines an eager mode.

```text
knot validate config/app.knot
```

Eager validation loads every import and resolves every reference and inheritance chain. It reports all errors it finds, not only the first, and exits with a non-zero status if there are any.

Libraries should offer the same check at load time, for example as a `validate: true` option, so applications can fail at startup instead of on first access.

## Circular Reference Detection

Circular references are invalid.

```ini
[a]
value: @b.value

[b]
value: @a.value
```

The resolver reports the complete dependency chain.

```text
KnotReferenceError: Circular reference detected

a.value
  -> b.value
  -> a.value

Location: config/app.knot:2
```

Circular references spanning multiple files are also detected.

---

# Validation and Error Handling

Knot uses strict validation to prevent ambiguous configuration behavior.

### Duplicate Properties

Invalid:

```ini
[database]
host: localhost
host: production.internal
```

Error:

```text
KnotSyntaxError: Duplicate property "host"

Section: database
Location: config/app.knot:3
```

### Duplicate Sections

Invalid:

```ini
[database]
host: localhost

[cache]
driver: redis

[database]
port: 3306
```

Error:

```text
KnotSyntaxError: Duplicate section "database"

First declared: config/app.knot:1
Location: config/app.knot:7
```

### Conflicting Types

Invalid:

```ini
[database]
credentials: production

[database.credentials]
username: root
```

A scalar property cannot simultaneously represent a configuration section.

### Invalid Literals

Invalid:

```ini
[app]
version: 1.2.3
enabled: True
```

Error:

```text
KnotSyntaxError: Invalid number "1.2.3"

Hint: Quote the value to use it as a string.
Location: config/app.knot:2
```

```text
KnotSyntaxError: Invalid literal "True"

Hint: Booleans and null are lowercase.
Location: config/app.knot:3
```

### Invalid Dates

Invalid:

```ini
[release]
published: 2026-02-30
```

Error:

```text
KnotSyntaxError: Invalid date "2026-02-30"

Hint: February 2026 has 28 days.
Location: config/app.knot:2
```

### Invalid References

Invalid:

```ini
[database]
host: @missing.host
```

Error when resolved, or from `knot validate`:

```text
KnotReferenceError: Reference not found

Reference: @missing.host
Location: config/app.knot:2
```

### Invalid Inheritance

Invalid:

```ini
[app]
workers: 8

[queue] < @app.workers
```

Error:

```text
KnotReferenceError: Cannot inherit from a non-object value

Reference: @app.workers (integer)
Location: config/app.knot:4
```

### Invalid Inline Values

Invalid:

```ini
[server]
hosts: [web1.local, web2.local]
```

Error:

```text
KnotSyntaxError: Unquoted string in inline array

Hint: Strings inside inline objects and arrays must be quoted.
Location: config/app.knot:2
```

---

# Security

Configuration references introduce additional security considerations.

Knot's reference resolver is designed around the following principles:

- Imports are limited to local `.knot` files.
- File access can be restricted to configured directories.
- Circular references are rejected.
- Maximum reference depth can be configured.
- Maximum file size and nesting depth can be enforced.
- Environment variable and secret resolution require explicit opt-in.
- Imported files are treated as untrusted input unless explicitly trusted.

Knot does not execute configuration values as code.

---

# Language Comparison

| Feature | Knot | TOML | YAML | JSON | HOCON |
|---|---|---|---|---|---|
| Human-readable | ✅ | ✅ | ✅ | ✅ | ✅ |
| Indentation-independent | ✅ | ✅ | Partial | ✅ | ✅ |
| Section headers | ✅ | ✅ | ❌ | ❌ | ❌ |
| `key: value` assignments | ✅ | ❌ | ✅ | ❌ | ✅ |
| Unquoted strings | ✅ | ❌ | ✅ | ❌ | ✅ |
| JSON-compatible values | ✅ | Partial | ✅ (1.2) | ✅ | ✅ |
| Native null | ✅ | ❌ | ✅ | ✅ | ✅ |
| Native dates | ✅ | ✅ | 1.1 only | ❌ | ❌ |
| Arrays of sections | ✅ | ✅ | ❌ | ❌ | ❌ |
| Property references | ✅ | ❌ | Aliases | ❌ | ✅ |
| Cross-file imports | ✅ | ❌ | ❌ | ❌ | ✅ |
| Environment variables | ✅ (opt-in) | ❌ | ❌ | ❌ | ✅ |
| Inheritance | ✅ | ❌ | Merge keys (1.1) | ❌ | Object merging |
| Duplicate keys are errors | ✅ | ✅ | Parser-dependent | Parser-dependent | ❌ (merged) |
| Circular-reference detection | ✅ | N/A | Alias handling | N/A | ✅ |

The comparison describes built-in language capabilities rather than features provided by external libraries or application frameworks.

## Why Not HOCON?

HOCON is the closest existing format. It already has unquoted strings, `${path}` substitutions, `include`, and JSON-superset values. Knot differs in four ways:

- **Section headers.** Knot uses TOML-style `[section]` headers and `[[array]]` sections instead of nested braces.
- **Duplicates are errors.** HOCON merges or overrides duplicate keys by design. Knot rejects them, so each value is defined in exactly one place.
- **Explicit inheritance.** HOCON merges any objects assigned to the same path. Knot inherits only where a section declares `<`.
- **Named imports.** HOCON's `include` merges another file into the current one. Knot's `@import` binds the file to a name, so every reference shows which file it reads from.

## Programmable Configuration Languages

Pkl, CUE, Jsonnet, and Dhall add functions, types, or logic to configuration. Knot stays a data format: references and inheritance are its only evaluation.

---

# Design Philosophy

Knot follows five principles.

**1. Configuration should be readable.**

Developers should be able to understand a configuration file without learning complex syntax rules.

**2. Whitespace should not define structure.**

Indentation is a formatting preference, not a parsing requirement.

**3. Configuration should be reusable.**

References and inheritance should eliminate unnecessary duplication.

**4. Types should be predictable.**

Values should retain their original types through parsing and reference resolution.

**5. Configuration should not execute code.**

Knot is a declarative data format, not a scripting language.

---

# Project Status

**Status: Proposal / Early Design**

Knot is currently a proposed configuration language.

The syntax, reference model, and inheritance behavior described here represent the initial design and may change before the first implementation.

## Proposed Roadmap

### Phase 1 — Language Specification

- [ ] Formal grammar
- [ ] Comments and unquoted-string rules
- [ ] Scalar data types
- [ ] Dates and times
- [ ] Keys and quoted keys
- [ ] Section headers and nested sections
- [ ] Inline objects and arrays
- [ ] Arrays of sections
- [ ] Multiline strings
- [ ] Error handling specification

### Phase 2 — Reference System

- [ ] Local property references
- [ ] Section references
- [ ] Reference paths and array indexing
- [ ] Chained references
- [ ] Imports (`@import`)
- [ ] Lazy resolution
- [ ] Eager validation
- [ ] Circular-reference detection
- [ ] Reference caching

### Phase 3 — Advanced Features

- [ ] Section inheritance
- [ ] Configurable file access restrictions
- [ ] Environment variables (`@env`)
- [ ] Optional secret resolvers
- [ ] Configuration schema validation

### Phase 4 — Tooling

- [ ] Reference parser implementation
- [ ] Command-line validator (`knot validate`)
- [ ] Configuration formatter
- [ ] JSON-to-Knot converter
- [ ] Syntax highlighting
- [ ] Language server support
- [ ] Parser implementations for additional languages

---

# Contributing

Contributions, ideas, and feedback are welcome.

The initial focus is defining a predictable, portable language specification before introducing implementation-specific behavior.

# License

The specification text is licensed under [CC BY 4.0](LICENSE). Reference implementations will be licensed under the [MIT License](https://opensource.org/licenses/MIT).
