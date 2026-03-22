# abap-fabric — Project Plan

## Vision

An ontology-driven distributed SDK enabling ABAP systems to call remote services using
locally generated stubs, and enabling remote systems to call ABAP the same way.
Protobuf is the serialization wire. The enterprise ontology is the source of truth for
all service/message contracts.

**Target integration:** ABAP (on-prem S/4HANA, NS2 secure cloud) ↔ Apriso and other
enterprise services, with stubs generated for both sides from the ontology.

---

## Repository Structure

```
netsemantics/abap-fabric        ABAP runtime codec (this repo, abapGit-compatible)
netsemantics/protoc-gen-abap    Go-based protoc plugin (generates ABAP stubs)
```

---

## Architecture

```
Enterprise Ontology  (source of truth — Phase 3)
        │
        ▼
  .proto definitions
        │
        ├──► protoc-gen-abap (Go)  ──► ABAP stubs + zcl_*_codec classes
        │
        └──► protoc-gen-* (existing tools)  ──► Python / Go / Java stubs
                                                  (e.g. Apriso side)

ABAP Runtime (this repo)
  └── zcl_protobuf_codec    wire encode/decode (varint, fixed32/64, delimited)
  └── zcx_protobuf_error    typed exceptions (MALFORMED, TRUNCATED, etc.)
  └── zif_protobuf_message  stable consumer interface

Transport layer (Phase 3+)
  └── HTTP/gRPC bridge between ABAP and remote stubs
```

---

## Phases

### Phase 1 — Hardened proto2 wire codec  ✦ current focus

Goal: a correct, tested, safe-by-default ABAP serialization runtime that all
later layers can depend on.

#### Milestones

- [ ] **1.1 — CI tooling baseline**
  - abaplint configured (v750 syntax baseline)
  - open-abap transpiler config
  - GitHub Actions: lint + unit test on every push
  - Gate: `ASSERT 1 = 'todo'` blocks merge to main

- [ ] **1.2 — Core codec (`zcl_protobuf_codec`)**
  - Varint encode/decode (copy + harden from reference fork)
  - Fixed64 encode/decode (correct IEEE 754 double)
  - Fixed32 encode/decode (correct IEEE 754 float — *not* delegating to double)
  - Zig-zag encode/decode for `sint32` / `sint64`
  - Length-delimited (bytes/string) encode/decode
  - Field tag encode/decode (field number + wire type)
  - Unknown-field skipping (wire-type-aware — no asserts)

- [ ] **1.3 — Typed exception model (`zcx_protobuf_error`)**
  - Categories: `MALFORMED`, `TRUNCATED`, `INVALID_WIRE_TYPE`,
    `UNKNOWN_ENUM_VALUE`, `REQUIRED_MISSING`, `SIZE_LIMIT`
  - Replace all `ASSERT` / `ASSERT 1 = 'todo'` in codec with raises

- [ ] **1.4 — Consumer interface (`zif_protobuf_message`)**
  - `serialize( ) RETURNING xstring`
  - `deserialize( iv_bytes TYPE xstring )`

- [ ] **1.5 — Golden-vector tests**
  - Wire vectors for: int32/int64/uint32/uint64, sint32/sint64,
    fixed32/fixed64, float/double, bytes/string, embedded messages,
    packed repeated
  - Negative tests: truncated input, invalid varint, mismatched wire type
  - Interop: encode in ABAP, decode in reference impl (and vice versa)

#### Reference material
- Fork at `github.com/netsemantics/abap-protobuf` — varint/fixed64 codec
  and ABAP type mappings are useful reference; do not copy asserts or parser.
- `protobuf.dev/programming-guides/encoding/` — wire format specification

---

### Phase 2 — External codegen + proto3

Goal: replace in-ABAP text parsing with a proper protoc plugin; add proto3 support.

#### Milestones

- [ ] **2.1 — `protoc-gen-abap` plugin (Go)**
  - New repo: `netsemantics/protoc-gen-abap`
  - Reads `CodeGeneratorRequest` (FileDescriptorSet from protoc)
  - Generates `ZIF_<Message>` type interfaces
  - Generates `ZCL_<Message>_CODEC` serialize/deserialize classes
  - ABAP naming rules: 30-char limit, uppercase, hash long identifiers
  - Output follows abapGit `src/` layout

- [ ] **2.2 — Proto3 support**
  - Field presence rules (proto3 implicit presence)
  - `oneof` fields
  - `map<K,V>` fields
  - `reserved` field numbers

- [ ] **2.3 — Well-known types runtime helpers**
  - `google.protobuf.Timestamp` ↔ ABAP timestamp
  - `google.protobuf.Duration`
  - `google.protobuf.Any` (type URL + packing)
  - Wrapper types (BoolValue, StringValue, etc.)

- [ ] **2.4 — ProtoJSON mapping**
  - Encode/decode per ProtoJSON rules
  - Well-known type JSON encodings (Timestamp as RFC 3339, etc.)
  - Interop tests against reference JSON mapping

- [ ] **2.5 — Buf integration**
  - `buf.yaml` + `buf.gen.yaml` in repo
  - CI gates: `buf lint` + `buf breaking` on every PR

---

### Phase 3 — Ontology integration + transport

Goal: wire the enterprise ontology as the schema source of truth; add transport layer
for real ABAP ↔ remote service calls.

#### Milestones

- [ ] **3.1 — Ontology → proto bridge**
  - Map ontology concepts (entities, relationships, services) to proto
    messages and service definitions
  - Pipeline: ontology export → `.proto` files → protoc pipeline
  - First target: Apriso integration service definition

- [ ] **3.2 — Transport layer**
  - HTTP client adapter in ABAP for calling gRPC/REST endpoints
  - Service stub base class (generated by `protoc-gen-abap`)
  - Authentication / header injection hooks

- [ ] **3.3 — Inbound stubs (remote → ABAP)**
  - ABAP as service provider: generated handler classes
  - ICF handler or RFC wrapper depending on transport choice

- [ ] **3.4 — ABAP Cloud compatibility pass**
  - Audit all runtime classes against released API whitelist
  - Document restricted vs full language version requirements
  - ADT-compatible package organization

---

## ABAP Type Mapping (reference)

| Proto type     | ABAP type   | Notes                                      |
|----------------|-------------|---------------------------------------------|
| `int32`        | `i`         |                                             |
| `int64`        | `int8`      |                                             |
| `uint32`       | `int8`      | Wider than needed; revisit                  |
| `uint64`       | `int8`      | Unsigned semantics via bit pattern          |
| `sint32`       | `i`         | Zig-zag encoded on wire                     |
| `sint64`       | `int8`      | Zig-zag encoded on wire                     |
| `fixed32`      | `int4`      | Always 4 bytes                              |
| `fixed64`      | `int8`      | Always 8 bytes                              |
| `sfixed32`     | `i`         |                                             |
| `sfixed64`     | `int8`      |                                             |
| `float`        | `f`         | Encode as fixed32; decode rounding accepted |
| `double`       | `f`         | ABAP `f` is 64-bit IEEE 754                 |
| `bool`         | `abap_bool` |                                             |
| `string`       | `string`    | UTF-8                                       |
| `bytes`        | `xstring`   |                                             |
| `enum`         | `i`         | Generated constant interface                |
| message (nested)| `REF TO zif_<name>` |                                  |
| repeated       | `STANDARD TABLE OF ...` |                               |

---

## CI Pipeline (target state)

```
push / PR
  │
  ├── abaplint          ABAP static analysis (v750 baseline)
  ├── no-todo gate      fail if ASSERT 1 = 'todo' exists in src/
  ├── open-abap unit    transpile + run ABAP Unit tests
  ├── buf lint          proto schema quality (Phase 2+)
  └── buf breaking      schema compatibility gate (Phase 2+)
```

---

## Key Decisions Made

| Decision | Choice | Reason |
|---|---|---|
| ABAP syntax baseline | v750 | Widest on-prem S/4HANA compatibility |
| Target environment | On-prem S/4HANA (NS2 secure cloud) | Cloud compatibility deferred to Phase 3 |
| Codegen approach | External protoc plugin (Go) | Avoids re-implementing proto grammar in ABAP; aligns with protoc ecosystem |
| Error handling | Typed exceptions (`zcx_protobuf_error`) | No `ASSERT` in production code |
| Schema truth | Enterprise ontology → proto (Phase 3) | Ontology work runs in parallel |
| Repo structure | Two repos: runtime + plugin | Separate ABAP and Go lifecycles |
| Proto version order | proto2 first, proto3 in Phase 2 | Gets working codec faster |

---

## Open Questions

- Transport protocol for ABAP ↔ Apriso: gRPC over HTTP/2, REST+JSON, or RFC?
- First concrete Apriso service to demonstrate end-to-end (drives Phase 3.1 scope)
- ABAP package naming convention for generated classes

---

*Last updated: 2026-03-22*
