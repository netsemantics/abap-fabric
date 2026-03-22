# abap-fabric

An ontology-driven distributed SDK for ABAP — enabling ABAP systems to call remote services using local-feeling generated stubs, and enabling remote systems to call ABAP the same way.

## Architecture

```
Enterprise Ontology (source of truth)
        │
        ▼
  Proto definitions
        │
        ├──── protoc-gen-abap (Go plugin) ──► ABAP stubs + runtime
        │
        └──── protoc-gen-* (existing tools) ──► Python / Go / Java stubs
```

## Components

- **`src/`** — ABAP runtime codec (abapGit-compatible)
- **[protoc-gen-abap](https://github.com/netsemantics/protoc-gen-abap)** — Go-based protoc plugin for generating ABAP stubs

## Status

Early development. Phase 1: hardened proto2 wire codec.

## License

MIT
