# jsonschema

[![ci](https://github.com/Stranger6667/jsonschema-rs/workflows/ci/badge.svg)](https://github.com/Stranger6667/jsonschema-rs/actions)
[![codecov](https://codecov.io/gh/Stranger6667/jsonschema-rs/branch/master/graph/badge.svg)](https://codecov.io/gh/Stranger6667/jsonschema-rs)
[![Crates.io](https://img.shields.io/crates/v/jsonschema.svg)](https://crates.io/crates/jsonschema)
[![docs.rs](https://docs.rs/jsonschema/badge.svg)](https://docs.rs/jsonschema/)
[![gitter](https://img.shields.io/gitter/room/Stranger6667/jsonschema-rs.svg)](https://gitter.im/Stranger6667/jsonschema-rs)

A JSON Schema validator implementation. It compiles schema into a validation tree to have validation as fast as possible.

Supported drafts:

- Draft 7 (except optional `idn-hostname.json` test case)
- Draft 6
- Draft 4 (except optional `bignum.json` test case)

```toml
# Cargo.toml
jsonschema = "0.13"
```

To validate documents against some schema and get validation errors (if any):

```rust
use jsonschema::{Draft, JSONSchema};
use serde_json::json;

fn main() {
    let schema = json!({"maxLength": 5});
    let instance = json!("foo");
    let compiled = JSONSchema::compile(&schema)
        .expect("A valid schema");
    let result = compiled.validate(&instance);
    if let Err(errors) = result {
        for error in errors {
            println!("Validation error: {}", error);
            println!(
                "Instance path: {}", error.instance_path
            );
        }
    }
}
```

Each error has an `instance_path` attribute that indicates the path to the erroneous part within the validated instance.
It could be transformed to JSON Pointer via `.to_string()` or to `Vec<String>` via `.into_vec()`.

If you only need to know whether document is valid or not (which is faster):

```rust
use jsonschema::is_valid;
use serde_json::json;

fn main() {
    let schema = json!({"maxLength": 5});
    let instance = json!("foo");
    assert!(is_valid(&schema, &instance));
}
```

Or use a compiled schema (preferred):

```rust
use jsonschema::{Draft, JSONSchema};
use serde_json::json;

fn main() {
    let schema = json!({"maxLength": 5});
    let instance = json!("foo");
    // Draft is detected automatically
    // with fallback to Draft7
    let compiled = JSONSchema::compile(&schema)
        .expect("A valid schema");
    assert!(compiled.is_valid(&instance));
}
```

## Output styles

`jsonschema` supports `basic` & `flag` output styles from Draft 2019-09, so you can serialize the validation results with `serde`:

```rust
use jsonschema::{Draft, Output, BasicOutput, JSONSchema};

fn main() {
    let schema_json = serde_json::json!({
        "title": "string value",
        "type": "string"
    });
    let instance = serde_json::json!{"some string"};
    let schema = JSONSchema::options()
        .compile(&schema_json)
        .expect("A valid schema");
    
    let output: BasicOutput = schema.apply(&instance).basic();
    let output_json = serde_json::to_value(output)
        .expect("Failed to serialize output");
    
    assert_eq!(
        output_json, 
        serde_json::json!({
            "valid": true,
            "annotations": [
                {
                    "keywordLocation": "",
                    "instanceLocation": "",
                    "annotations": {
                        "title": "string value"
                    }
                }
            ]
        })
    );
}
```

## Status

This library is functional and ready for use, but its API is still evolving to the 1.0 API.

## Bindings

- Python - See the `./bindings/python` directory
- Ruby - a [crate](https://github.com/driv3r/rusty_json_schema) by @driv3r
- NodeJS - a [package](https://github.com/ahungrynoob/jsonschema) by @ahungrynoob

## Performance

There is a comparison with other JSON Schema validators written in Rust - `jsonschema_valid==0.4.0` and `valico==3.6.0`.

Test machine i8700K (12 cores), 32GB RAM.

Input values and schemas:

- [Zuora](https://github.com/APIs-guru/openapi-directory/blob/master/APIs/zuora.com/2021-04-23/openapi.yaml) OpenAPI schema (`zuora.json`). Validated against [OpenAPI 3.0 JSON Schema](https://github.com/OAI/OpenAPI-Specification/blob/main/schemas/v3.0/schema.json) (`openapi.json`).
- [Kubernetes](https://raw.githubusercontent.com/APIs-guru/openapi-directory/master/APIs/kubernetes.io/v1.10.0/swagger.yaml) Swagger schema (`kubernetes.json`). Validated against [Swagger JSON Schema](https://github.com/OAI/OpenAPI-Specification/blob/main/schemas/v2.0/schema.json) (`swagger.json`).
- Canadian border in GeoJSON format (`canada.json`). Schema is taken from the [GeoJSON website](https://geojson.org/schema/FeatureCollection.json) (`geojson.json`).
- Concert data catalog (`citm_catalog.json`). Schema is inferred via [infers-jsonschema](https://github.com/Stranger6667/infers-jsonschema) & manually adjusted (`citm_catalog_schema.json`).
- `Fast` is taken from [fastjsonschema benchmarks](https://github.com/horejsek/python-fastjsonschema/blob/master/performance.py#L15) (`fast_schema.json`, `fast_valid.json` and `fast_invalid.json`).

| Case           | Schema size | Instance size |
| -------------- | ----------- | ------------- |
| OpenAPI        | 18 KB       | 4.5 MB        |
| Swagger        | 25 KB       | 3.0 MB        |
| Canada         | 4.8 KB      | 2.1 MB        |
| CITM catalog   | 2.3 KB      | 501 KB        |
| Fast (valid)   | 595 B       | 55 B          |
| Fast (invalid) | 595 B       | 60 B          |

Here is the average time for each contender to validate. Ratios are given against compiled `JSONSchema` using its `validate` method. The `is_valid` method is faster, but gives only a boolean return value:

| Case           | jsonschema_valid        | valico                  | jsonschema (validate) | jsonschema (is_valid)  |
| -------------- | ----------------------- | ----------------------- | --------------------- | ---------------------- |
| OpenAPI        |                   - (1) |                   - (1) |              5.231 ms |   4.712 ms (**x0.90**) |
| Swagger        |                   - (2) |  92.861 ms (**x12.27**) |              7.565 ms |   4.954 ms (**x0.65**) |
| Canada         |  35.773 ms (**x29.06**) | 152.84 ms (**x124.15**) |              1.231 ms |   1.233 ms (**x1.00**) |
| CITM catalog   |    5.215 ms (**x1.92**) |   14.555 ms (**x5.38**) |              2.703 ms |  576.38 us (**x0.21**) |
| Fast (valid)   |     2.14 us (**x3.32**) |     3.53 us (**x5.49**) |             642.95 ns |  107.34 ns (**x0.16**) |
| Fast (invalid) |   380.58 ns (**x0.47**) |     3.64 us (**x4.54**) |             800.74 ns |    7.19 ns (**x0.01**) |

Notes:

1. `jsonschema_valid` and `valico` do not handle valid path instances matching the `^\\/` regex.

2. `jsonschema_valid` fails to resolve local references (e.g. `#/definitions/definitions`).

You can find benchmark code in `benches/jsonschema.rs`, Rust version is `1.56`.

## Support

If you have anything to discuss regarding this library, please, join our [gitter](https://gitter.im/Stranger6667/jsonschema-rs)!
