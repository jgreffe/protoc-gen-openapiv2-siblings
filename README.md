# OpenAPI Siblings Issue Reproduction

This minimal Go project demonstrates the **no-$ref-siblings** and **description-duplication** issues that occur when generating OpenAPI v2 specifications from Protocol Buffer definitions using [protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway/tree/main/protoc-gen-openapiv2).

## TL;DR

### 1. Install `go` and `task`
 - https://go.dev/doc/install
 - https://taskfile.dev/docs/installation

### 2. Execute the following command
 - install dependencies and tools
 - generate code and OpenAPI spec
 - inspect the generated OpenAPI spec for issues

   ```bash
   task install-tools generate inspect
   ```

### 3. See errors

```
✗ 3 errors for no-$ref-siblings
● 4 issues for description-duplication
Quality Score: 53/100 [🤒]
```

## Issues Demonstrated

### 1. No-$ref-siblings Issue

OpenAPI v2 specification doesn't allow a schema to have both a `$ref` and sibling properties at the same level. This commonly occurs with protobuf `oneof` fields, which need to reference types while also adding discriminator information.

**What causes it:**
- Proto `oneof` fields that contain complex message types
- The generator needs to inline the reference and add properties

**Example from this project:**
See `ItemWithChoice` message in `proto/common.proto` which has a `oneof choice` field with nested message types.

### 2. Description Duplication

When the same message type is used in multiple places (request/response, nested fields, arrays), the OpenAPI generator may duplicate the description text across multiple schema definitions instead of properly reusing schema references.

**What causes it:**
- Repeated message fields
- Same message type used in multiple request/response messages
- Nested message hierarchies

## Project Structure

```
protoc-gen-openapiv2-siblings/
├── proto/                  # Protocol Buffer definitions
│   ├── common.proto       # Common messages with oneof (causes ref-siblings)
│   └── service.proto      # Service definition with HTTP annotations
├── gen/                   # Generated code (gitignored)
│   ├── go/               # Generated Go code
│   └── openapi/          # Generated OpenAPI specs
├── build/                # Build tools and dependencies (gitignored)
│   ├── google/api/       # Google API proto files
│   └── grpc-gateway/     # grpc-gateway proto files
├── openapi_config.yaml   # OpenAPI generation configuration
├── Taskfile.yml          # Task definitions for code generation
├── go.mod                # Go module definition
├── go.sum                # Go dependencies checksums
└── README.md             # This file
```

## Prerequisites

- Go 1.23+ (tested with Go 1.25)
- Task (Go task runner): Install with `go install github.com/go-task/task/v3/cmd/task@latest`
- Optionally `svn` for faster Google API proto download (falls back to git if not available)

## Usage

### 1. Install Dependencies and Tools

```bash
task install-tools
```

This will:
- Verify `protoc` compiler is installed (install manually if needed: `brew install protobuf` on macOS)
- Install all necessary protoc plugins (protoc-gen-go, protoc-gen-go-grpc, protoc-gen-grpc-gateway, protoc-gen-openapiv2)
- Fetch Google API proto files (via `svn` or git clone)
- Fetch grpc-gateway proto files for OpenAPI annotations

### 2. Generate Code and OpenAPI Spec

```bash
task generate
```

This generates:
- Go code from proto definitions
- gRPC service code
- gRPC-gateway reverse proxy code
- **OpenAPI v2 specification** (where the issues appear)

The OpenAPI generation uses configuration from `openapi_config.yaml` which sets the API title, description, version, and other metadata.

### 3. Inspect the Issues

The project uses [vacuum](https://quobix.com/vacuum/), a powerful OpenAPI linter, to validate and inspect the generated specification:

```bash
task inspect
```

This will automatically:
1. Install vacuum if not already installed
2. Run validation showing errors, warnings, and info messages
3. Highlight the specific issues like `no-$ref-siblings` and `description-duplication`

You can also manually examine the generated file:

```bash
cat gen/openapi/api.swagger.json
```

Look for:

#### No-$ref-siblings Example:
```json
{
  ...
  "items": {
    "type": "object",
    "$ref": "#/definitions/demoItemWithChoice"
  },
  ...
}
```

#### Description Duplication Example:
```json
{
  "definitions": {
    "ItemWithChoiceInRequest": {
      "description": "Description of the item - this description may get duplicated",
      "properties": {...}
    },
    "ItemWithChoiceInResponse": {
      "description": "Description of the item - this description may get duplicated",
      "properties": {...}
    }
  }
}
```