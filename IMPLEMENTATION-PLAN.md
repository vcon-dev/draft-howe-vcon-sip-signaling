# Reference Implementation Plan: vCon SIP Signaling Extension

This document outlines the plan for building reference implementations of the
**draft-howe-vcon-sip-signaling** extension across major SIP libraries,
PBX/switching platforms, and CPaaS providers.

---

## Goals

1. **Prove the spec** -- Demonstrate that the extension is implementable across
   diverse SIP stacks and identify any ambiguities early.
2. **Accelerate adoption** -- Give developers ready-to-use libraries so they
   can produce and consume vCons with SIP signaling data immediately.
3. **Cover the ecosystem** -- Span open-source SIP servers (FreeSWITCH,
   Asterisk, Opal/Opal-based), CPaaS platforms (Twilio, Infobip, Vonage), and
   standalone SIP libraries (OPAL, PJSIP, JsSIP).

---

## Architecture Overview

Each reference implementation follows a layered design:

```
┌──────────────────────────────────────────────────┐
│  Platform / CPaaS Adapter                        │
│  (FreeSWITCH ESL, Asterisk ARI, Twilio SDK, …)  │
├──────────────────────────────────────────────────┤
│  vcon-sip-signaling Core Library                 │
│  (Shared data model, builders, validators)       │
├──────────────────────────────────────────────────┤
│  vCon Core (JSON serialization, signing, …)      │
└──────────────────────────────────────────────────┘
```

**Core library** (Python + TypeScript) is platform-agnostic and handles:
- Building Party objects with `sip_contact`, `sip_user_agent`, `sip_display_name`
- Building Dialog objects with `sip_call_id`, `sip_from_tag`, `sip_to_tag`, `sip_cseq`
- Creating attachments for all 13 purpose values
- SIP message trace construction
- STIR/SHAKEN data (certificates, verification reports, extended PASSporTs)
- Credential redaction (Authorization, Proxy-Authorization, WWW-Authenticate)
- JSON Schema validation against the spec's schemas
- Minimal-producer vs full-producer profiles

**Platform adapters** sit on top and translate platform-specific events/APIs
into core library calls.

---

## Implementation 1: Core Library (Python)

**Directory:** `implementations/python/vcon_sip_signaling/`

### Modules

| Module | Responsibility |
|--------|----------------|
| `models.py` | Pydantic data models for Party extensions, Dialog extensions, all attachment types, SIP message trace, STIR verification report |
| `builders.py` | `MinimalProducer` and `FullProducer` builder classes that construct a complete vCon dict with the sip-signaling extension |
| `redaction.py` | Strip/redact sensitive SIP headers (Authorization, Proxy-Authorization, WWW-Authenticate, optionally Via/Record-Route/Path) |
| `validators.py` | Validate a vCon dict against the JSON Schema fragments defined in Section 13 of the draft |
| `sip_parser.py` | Parse raw SIP messages (wire format `message/sip`) into structured header dicts; extract Call-ID, tags, CSeq, Contact, User-Agent |
| `stir.py` | Helper for PASSporT handling: compact JWS ↔ JSON serialization, certificate chain packaging, verification report construction |
| `constants.py` | All 13 purpose values, extension token `"sip-signaling"`, media types |

### Key Classes

```python
class SipVconBuilder:
    """Fluent builder for constructing vCons with sip-signaling extension."""
    def add_party(tel, sip, name, stir=None, sip_contact=None, ...) -> int
    def set_dialog(call_id, from_tag=None, to_tag=None, cseq=None, ...)
    def attach_sip_message(purpose, raw_message, party, timestamp, ...)
    def attach_message_trace(messages: list[TraceMessage])
    def attach_sdp(sdp_body, party, timestamp)
    def attach_headers(headers_dict, party)
    def attach_stir_certificate(pem_chain, party)
    def attach_verification_report(report: VerificationReport, party)
    def build() -> dict  # Returns the complete vCon JSON
    def build_minimal() -> dict  # Only INVITE + response + call_id + stir
```

### Tasks

1. Set up `pyproject.toml` with dependencies: `pydantic`, `jsonschema`, `cryptography` (for STIR cert handling)
2. Implement `models.py` with all data classes matching Section 13 schemas
3. Implement `sip_parser.py` -- parse `message/sip` wire format into headers/body
4. Implement `redaction.py` -- header stripping per Section 9
5. Implement `builders.py` -- MinimalProducer and FullProducer
6. Implement `validators.py` -- JSON Schema validation
7. Implement `stir.py` -- PASSporT and certificate helpers
8. Write unit tests reproducing both spec examples (Section 12)
9. Write integration test that round-trips: build vCon → validate → parse back

---

## Implementation 2: Core Library (TypeScript/Node.js)

**Directory:** `implementations/typescript/`

Mirrors the Python core in TypeScript for the Node.js / browser ecosystem.

### Modules

| Module | Responsibility |
|--------|----------------|
| `src/models.ts` | TypeScript interfaces and Zod schemas for all extension types |
| `src/builder.ts` | `SipVconBuilder` class (fluent API) |
| `src/redaction.ts` | SIP header redaction |
| `src/validator.ts` | AJV-based JSON Schema validation |
| `src/sip-parser.ts` | SIP wire format parser |
| `src/stir.ts` | STIR/SHAKEN helpers |
| `src/constants.ts` | Purpose values, media types, extension token |

### Tasks

1. Set up `package.json` with `zod`, `ajv`, `jose` (for JWT/JWS)
2. Define TypeScript interfaces matching spec schemas
3. Implement builder with same fluent API as Python
4. Implement SIP parser and redaction
5. Implement validator
6. Write test suite mirroring Python tests

---

## Implementation 3: FreeSWITCH Adapter

**Directory:** `implementations/freeswitch/`
**Language:** Python (via ESL -- Event Socket Library)

### How It Works

FreeSWITCH exposes SIP signaling via its Event Socket Layer. The adapter
listens for channel events and constructs vCons from them.

### Integration Points

| FreeSWITCH Source | vCon Target |
|-------------------|-------------|
| `variable_sip_call_id` | `dialog.sip_call_id` |
| `variable_sip_from_tag` | `dialog.sip_from_tag` |
| `variable_sip_to_tag` | `dialog.sip_to_tag` |
| `variable_sip_cseq` | `dialog.sip_cseq` |
| `variable_sip_contact_uri` | `party.sip_contact` |
| `variable_sip_user_agent` | `party.sip_user_agent` |
| `variable_sip_from_display` | `party.sip_display_name` |
| `variable_caller_id_number` | `party.tel` |
| `variable_sip_req_uri` | `party.sip` |
| `CHANNEL_CREATE` / `CHANNEL_ANSWER` / `CHANNEL_HANGUP` events | Attachment timestamps |
| `sofia::profile::sip-trace` (via `siptrace` module) | `sip-message-trace` attachment |
| `variable_sip_Identity` (STIR header) | `party.stir` + STIR attachments |
| Recording file path from `RECORD_START`/`RECORD_STOP` | `dialog.body` (recording) |

### Modules

| Module | Responsibility |
|--------|----------------|
| `freeswitch_adapter.py` | ESL connection, event subscription, event-to-vCon mapping |
| `sip_trace_collector.py` | Collects SIP trace events and builds `sip-message-trace` |
| `stir_extractor.py` | Extracts STIR/SHAKEN data from FreeSWITCH variables |
| `config.py` | Producer profile (minimal/full), redaction settings, output destination |
| `main.py` | CLI entry point: connect to FreeSWITCH, listen, produce vCons |

### Tasks

1. Implement ESL connection and event listener
2. Map FreeSWITCH channel variables to Party/Dialog extensions
3. Implement SIP trace collection via `mod_sofia` trace events
4. Extract STIR data from `Identity` header variable
5. Produce minimal and full vCons using the core library
6. Write integration tests with recorded ESL event sequences
7. Document FreeSWITCH configuration (`mod_sofia` siptrace, recording setup)

---

## Implementation 4: Asterisk Adapter

**Directory:** `implementations/asterisk/`
**Language:** Python (via ARI -- Asterisk REST Interface and AMI)

### Integration Points

| Asterisk Source | vCon Target |
|-----------------|-------------|
| `PJSIP_HEADER(read,Call-ID)` | `dialog.sip_call_id` |
| `SIP_HEADER(From)` tag parsing | `dialog.sip_from_tag` |
| `SIP_HEADER(To)` tag parsing | `dialog.sip_to_tag` |
| `SIP_HEADER(CSeq)` parsing | `dialog.sip_cseq` |
| `SIP_HEADER(Contact)` | `party.sip_contact` |
| `SIP_HEADER(User-Agent)` | `party.sip_user_agent` |
| `CALLERID(name)` | `party.sip_display_name` |
| `CALLERID(num)` / `CALLERID(all)` | `party.tel` |
| ARI `StasisStart` / `StasisEnd` | Dialog timestamps |
| `res_pjsip_logger` packet captures | `sip-message-trace` attachment |
| `STIR_SHAKEN()` function results | STIR/verification data |
| MixMonitor / recording | `dialog.body` |

### Modules

| Module | Responsibility |
|--------|----------------|
| `ari_adapter.py` | ARI WebSocket connection, Stasis event handling |
| `ami_adapter.py` | AMI connection for SIP packet logging |
| `sip_header_extractor.py` | Parse SIP headers from PJSIP channel variables |
| `stir_shaken_extractor.py` | Map `res_stir_shaken` verification results to STIR attachments |
| `config.py` | Configuration for producer profile, channels, output |
| `main.py` | CLI entry point |

### Tasks

1. Implement ARI WebSocket event listener
2. Map PJSIP channel variables to Party/Dialog extensions
3. Implement AMI-based SIP logging for message trace
4. Map `res_stir_shaken` verification results to verification report schema
5. Produce vCons using the core library
6. Write tests with recorded ARI/AMI event sequences
7. Document Asterisk configuration (`pjsip.conf`, `stir_shaken.conf`)

---

## Implementation 5: Twilio CPaaS Adapter

**Directory:** `implementations/cpaas/twilio/`
**Language:** Python + TypeScript

### How It Works

Twilio doesn't expose raw SIP signaling to most users, but does provide:
- Call resource metadata via REST API
- SIP headers via `X-` headers on TwiML webhooks
- SIP trunking (Elastic SIP) with full SIP access
- SHAKEN attestation via the `StirStatus` / `StirVerstat` parameters

### Integration Points

| Twilio Source | vCon Target |
|---------------|-------------|
| `Call.sid` | `dialog.sip_call_id` (mapped; not actual SIP Call-ID) |
| `Call.from_` / `Call.to` | `party.tel` |
| `Call.caller_name` | `party.sip_display_name` |
| `SipHeader` (TwiML) | Individual SIP headers for extraction |
| `StirVerstat` webhook param | `stir-verification-report` result mapping |
| `StirPassportToken` | `party.stir` |
| Elastic SIP Trunk INVITE | Full `message/sip` attachments |
| Recording URL | `dialog.body` via external reference |

### Mapping StirVerstat to Verification Report

| Twilio `StirVerstat` | Report `result` | Report `attestation` |
|----------------------|-----------------|----------------------|
| `TN-Validation-Passed-A` | `verified` | `A` |
| `TN-Validation-Passed-B` | `verified` | `B` |
| `TN-Validation-Passed-C` | `verified` | `C` |
| `TN-Validation-Failed` | `failed` | (from token if available) |
| `No-TN-Validation` | `no-signature` | N/A |

### Modules

| Module | Responsibility |
|--------|----------------|
| `webhook_handler.py` | Flask/FastAPI handler for Twilio voice webhooks; extracts SIP data from request params |
| `call_resource_adapter.py` | Uses Twilio REST API to enrich vCon with call metadata |
| `elastic_sip_adapter.py` | For Elastic SIP Trunking: full SIP message capture |
| `stir_mapper.py` | Maps `StirVerstat` / `StirPassportToken` to STIR attachments |
| `recording_linker.py` | Attaches Twilio recording URLs as external dialog references |

### Tasks

1. Implement webhook handler that captures SIP-related params
2. Implement Twilio REST API enrichment
3. Map `StirVerstat` values to STIR verification reports
4. Implement Elastic SIP Trunking adapter for full SIP access
5. Handle recording attachment via URL + content hash
6. Write tests with sample webhook payloads
7. Provide example TwiML application configuration

---

## Implementation 6: Infobip CPaaS Adapter

**Directory:** `implementations/cpaas/infobip/`
**Language:** Python

### How It Works

Infobip's Voice API provides call metadata via webhooks and REST API.
SIP trunking is available for direct SIP access.

### Integration Points

| Infobip Source | vCon Target |
|----------------|-------------|
| `callId` (webhook) | `dialog.sip_call_id` (mapped) |
| `from` / `to` (webhook) | `party.tel` |
| `callerIdName` | `party.sip_display_name` |
| Call establishment/hangup timestamps | Dialog `start` / `duration` |
| SIP Trunking INVITE/BYE | Full SIP message attachments |
| Recording download URL | `dialog.body` via external reference |
| CDR (Call Detail Record) API | Supplementary metadata |

### Modules

| Module | Responsibility |
|--------|----------------|
| `webhook_handler.py` | Infobip voice webhook receiver |
| `calls_api_adapter.py` | REST API client for call metadata enrichment |
| `sip_trunk_adapter.py` | SIP Trunking integration for full signaling |
| `recording_linker.py` | Recording URL attachment |

### Tasks

1. Implement Infobip voice webhook handler
2. Map Infobip call events to vCon Party/Dialog extensions
3. Implement SIP Trunking adapter
4. Implement CDR enrichment
5. Write tests with sample Infobip webhook payloads
6. Document Infobip application configuration

---

## Implementation 7: Vonage (Nexmo) CPaaS Adapter

**Directory:** `implementations/cpaas/vonage/`
**Language:** Python

### Integration Points

| Vonage Source | vCon Target |
|---------------|-------------|
| `conversation_uuid` / `uuid` | `dialog.sip_call_id` (mapped) |
| `from` / `to` | `party.tel` |
| Event webhooks (answered, completed) | Dialog timestamps |
| SIP Connect for SIP-native access | Full SIP message attachments |
| Recording URL | `dialog.body` via external reference |

### Modules

| Module | Responsibility |
|--------|----------------|
| `webhook_handler.py` | Vonage voice event webhook receiver |
| `sip_connect_adapter.py` | SIP Connect integration for full signaling |
| `recording_linker.py` | Recording URL attachment |

### Tasks

1. Implement Vonage voice event webhook handler
2. Map Vonage events to vCon structure
3. Implement SIP Connect adapter
4. Write tests with sample webhook payloads

---

## Implementation 8: PJSIP / OPAL Standalone Library Adapter

**Directory:** `implementations/pjsip/`
**Language:** Python (via PJSUA2 bindings) or C

For developers building custom SIP applications with PJSIP directly (not
through Asterisk/FreeSWITCH), this adapter hooks into PJSIP's transaction
and dialog layer.

### Integration Points

| PJSIP Source | vCon Target |
|--------------|-------------|
| `pjsip_dialog.call_id` | `dialog.sip_call_id` |
| `pjsip_dialog.local.info.tag` | `dialog.sip_from_tag` |
| `pjsip_dialog.remote.info.tag` | `dialog.sip_to_tag` |
| `pjsip_cseq_hdr` | `dialog.sip_cseq` |
| `pjsip_contact_hdr` | `party.sip_contact` |
| `pjsua_call_info.remote_info` | `party.sip_display_name` |
| Transaction callbacks (`on_tsx_state`) | SIP message trace events |
| `pjmedia_transport` SDP | `sip-sdp` attachment |

### Tasks

1. Implement PJSUA2 callback hooks for SIP dialog/transaction events
2. Build SIP message trace from transaction state changes
3. Extract SDP from media transport
4. Produce vCons using the core library
5. Write example application (simple call with vCon output)

---

## Cross-Cutting Concerns

### Test Strategy

Each implementation includes three levels of testing:

| Level | What | How |
|-------|------|-----|
| **Unit** | Core library models, builders, validators, parsers | Pytest / Jest with the two spec examples |
| **Integration** | Platform adapters with recorded event sequences | Replay recorded ESL/ARI/webhook payloads |
| **Conformance** | Validate output vCons against JSON Schema | Shared conformance test suite (JSON files) |

**Shared conformance suite** (`tests/conformance/`):
- `minimal_call.json` -- Expected vCon for a minimal two-party call
- `full_trace.json` -- Expected vCon for a full trace with STIR/SHAKEN
- `transfer_scenario.json` -- Call transfer via REFER
- `conference_scenario.json` -- Multi-party conference
- `failed_call.json` -- Call that receives 486 Busy / 404 Not Found
- `schema.json` -- Combined JSON Schema for validation

### Security Checklist (all implementations)

- [ ] Strip `Authorization` headers before storing
- [ ] Strip `Proxy-Authorization` headers before storing
- [ ] Strip `WWW-Authenticate` headers before storing
- [ ] Optionally redact `Via`, `Record-Route`, `Path` (configurable)
- [ ] Never store credentials or passwords in vCon output
- [ ] Support vCon signing for integrity protection
- [ ] Support vCon encryption for PII protection

### Documentation (per implementation)

Each adapter includes:
- `README.md` -- Setup, configuration, quick start
- Platform-specific configuration guide (e.g., FreeSWITCH `siptrace`, Asterisk `pjsip.conf`)
- Example output vCon (minimal and full)
- API reference (for library usage)

---

## Implementation Priority & Phases

### Phase 1: Foundation (Weeks 1-3)

| # | Task | Rationale |
|---|------|-----------|
| 1 | Python core library (`models`, `builders`, `validators`) | Everything depends on this |
| 2 | Conformance test suite | Validates all other implementations |
| 3 | SIP parser + redaction | Needed by all server-side adapters |

### Phase 2: Open-Source SIP Servers (Weeks 4-7)

| # | Task | Rationale |
|---|------|-----------|
| 4 | FreeSWITCH adapter | Most common open-source softswitch |
| 5 | Asterisk adapter | Most deployed open-source PBX |
| 6 | TypeScript core library | Enables browser/Node.js ecosystem |

### Phase 3: CPaaS Platforms (Weeks 8-11)

| # | Task | Rationale |
|---|------|-----------|
| 7 | Twilio adapter | Largest CPaaS platform |
| 8 | Infobip adapter | Major European/global CPaaS |
| 9 | Vonage adapter | Third major CPaaS with strong SIP support |

### Phase 4: Advanced & Polish (Weeks 12-14)

| # | Task | Rationale |
|---|------|-----------|
| 10 | PJSIP standalone adapter | For custom SIP applications |
| 11 | Call transfer scenarios (REFER) | Advanced SIP scenario coverage |
| 12 | Conference scenarios | Multi-party handling |
| 13 | End-to-end demo | FreeSWITCH call → vCon → validate → display |

---

## Directory Structure

```
implementations/
├── python/
│   └── vcon_sip_signaling/
│       ├── pyproject.toml
│       ├── src/
│       │   └── vcon_sip_signaling/
│       │       ├── __init__.py
│       │       ├── models.py
│       │       ├── builders.py
│       │       ├── redaction.py
│       │       ├── validators.py
│       │       ├── sip_parser.py
│       │       ├── stir.py
│       │       └── constants.py
│       └── tests/
│           ├── test_models.py
│           ├── test_builders.py
│           ├── test_redaction.py
│           ├── test_sip_parser.py
│           └── test_validators.py
├── typescript/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── models.ts
│   │   ├── builder.ts
│   │   ├── redaction.ts
│   │   ├── validator.ts
│   │   ├── sip-parser.ts
│   │   ├── stir.ts
│   │   └── constants.ts
│   └── tests/
│       └── ...
├── freeswitch/
│   ├── README.md
│   ├── freeswitch_adapter.py
│   ├── sip_trace_collector.py
│   ├── stir_extractor.py
│   ├── config.py
│   ├── main.py
│   └── tests/
├── asterisk/
│   ├── README.md
│   ├── ari_adapter.py
│   ├── ami_adapter.py
│   ├── sip_header_extractor.py
│   ├── stir_shaken_extractor.py
│   ├── config.py
│   ├── main.py
│   └── tests/
├── cpaas/
│   ├── twilio/
│   │   ├── README.md
│   │   ├── webhook_handler.py
│   │   ├── call_resource_adapter.py
│   │   ├── elastic_sip_adapter.py
│   │   ├── stir_mapper.py
│   │   ├── recording_linker.py
│   │   └── tests/
│   ├── infobip/
│   │   ├── README.md
│   │   ├── webhook_handler.py
│   │   ├── calls_api_adapter.py
│   │   ├── sip_trunk_adapter.py
│   │   ├── recording_linker.py
│   │   └── tests/
│   └── vonage/
│       ├── README.md
│       ├── webhook_handler.py
│       ├── sip_connect_adapter.py
│       ├── recording_linker.py
│       └── tests/
├── pjsip/
│   ├── README.md
│   ├── pjsua2_adapter.py
│   └── tests/
└── tests/
    └── conformance/
        ├── minimal_call.json
        ├── full_trace.json
        ├── transfer_scenario.json
        ├── conference_scenario.json
        ├── failed_call.json
        └── schema.json
```

---

## Open Questions & Decisions Needed

1. **CPaaS Call-ID mapping**: Twilio/Infobip/Vonage use platform-specific call
   IDs rather than SIP Call-IDs. Should adapters store the platform ID in
   `sip_call_id` or leave it empty and add a platform-specific field?
   *Recommendation*: Store the actual SIP Call-ID when available (e.g., via SIP
   Trunking), otherwise store the platform call ID with a note in metadata.

2. **Recording handling**: CPaaS recordings are typically available via URL, not
   inline. Should adapters always use external references (`url` + `content_hash`)
   for CPaaS recordings?
   *Recommendation*: Yes, use external references with content hash for
   integrity verification.

3. **STIR/SHAKEN availability**: STIR data is not uniformly available across
   platforms. Twilio provides `StirVerstat`; FreeSWITCH requires `mod_stir_shaken`;
   Asterisk has `res_stir_shaken`. Each adapter should document what STIR data
   is available and gracefully handle its absence.

4. **Real-time vs batch**: Should adapters produce vCons in real-time (on call
   completion) or support batch processing of CDRs?
   *Recommendation*: Primary mode is real-time on call completion. Batch mode
   can be added as an enhancement for CDR-based retroactive vCon generation.

5. **Which Python vCon library to build on**: Is there an existing Python vCon
   core library, or do we need to build one?
   *Recommendation*: Check for existing `py-vcon` or similar. If none exists,
   the core library should also implement basic vCon structure (parties, dialog,
   attachments, extensions array).
