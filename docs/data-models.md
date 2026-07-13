# Core data models

The canonical machine contract is [har-v1.schema.json](../schemas/har-v1.schema.json). Every document requires `apiVersion: "har/v1"` and stable `id`; aggregate documents additionally use positive integer `revision`. Unknown fields are allowed for forward-compatible readers, but consumers reject an unknown major `apiVersion`. Breaking change means `har/v2`; additive fields and enum additions are minor contract releases. IDs link `ProjectState → HardwareContext/model`, `ComponentInstance/Connection → BoardDescriptor/PinDefinition`, `ExperimentRun → ExperimentDefinition/artifact/evidence`, and reports/findings → evidence.

| Model family | Schema-required semantic fields | Example |
|---|---|---|
| ProjectState | lifecycle, hardwareModelId | `{"apiVersion":"har/v1","kind":"ProjectState","id":"p1","lifecycle":"ready_to_build","hardwareModelId":"m1"}` |
| BoardDescriptor / PinDefinition | adapter/FQBN/pins; name/capabilities | `{"apiVersion":"har/v1","kind":"BoardDescriptor","id":"uno","adapterId":"arduino-cli","platform":"arduino","fqbn":"arduino:avr:uno","pins":[]}` |
| ComponentInstance / Connection | driver/stateOrigin; endpoints/origin | `{"apiVersion":"har/v1","kind":"ComponentInstance","id":"u1","driverId":"sensor.ultrasonic.hc-sr04","stateOrigin":"user_reported"}` |
| FirmwareArtifact / HardwareContext | digest/board/status; project/revision/observations | `{"apiVersion":"har/v1","kind":"FirmwareArtifact","id":"a1","digest":"sha256:x","boardFqbn":"arduino:avr:uno","buildStatus":"compiled"}` |
| DiagnosticReport / Finding | findings; rule/confidence/evidence | `{"apiVersion":"har/v1","kind":"DiagnosticFinding","id":"f1","ruleId":"boot-loop","confidence":0.9,"evidenceIds":["e1"]}` |
| SerialObservation / UsbObservation | timestamp/data/source; event/fingerprint | `{"apiVersion":"har/v1","kind":"UsbObservation","id":"u1","timestamp":"2026-07-13T00:00:00Z","event":"attached","fingerprint":"vid:2341"}` |
| DriverDefinition | version/capabilities | `{"apiVersion":"har/v1","kind":"DriverDefinition","id":"d1","version":"1.0.0","capabilities":["distance_cm"]}` |
| ExperimentDefinition / Step / Run | version/steps; type/timeout; definition/status | `{"apiVersion":"har/v1","kind":"ExperimentRun","id":"r1","definitionId":"blink-led-v1","status":"running"}` |
| Human request / response | run/token/expiry; request/token/response | `{"apiVersion":"har/v1","kind":"HumanActionResponse","id":"hr1","requestId":"hq1","token":"opaque","response":{"confirmed":true}}` |
| Evidence / VerificationReport | digest/source/origin; verdict/evidence | `{"apiVersion":"har/v1","kind":"VerificationReport","id":"vr1","verdict":"passed","evidenceIds":["e1"]}` |
| ResourceReport / SimulationState | safety status/items; engine/status | `{"apiVersion":"har/v1","kind":"ResourceReport","id":"rr1","status":"warning","items":[]}` |
| RuntimeEvent / ErrorEnvelope | sequence/type/payload; code/category/stage/retry/evidence | `{"apiVersion":"har/v1","kind":"ErrorEnvelope","id":"x1","code":"FLASH_PORT_LOST","category":"flash_failure","message":"Port disappeared","stage":"flashing","retryable":true,"evidence":[]}` |

`stateOrigin` is mandatory wherever a state assertion is stored: `user_reported` is unverified user input; `runtime_observed` is a direct adapter capture; `inferred` is a reproducible derivation with cited evidence; `verified` is a successful assertion backed by named evidence and verification rule. These values never overwrite each other; context exposes both assertion and provenance.

## Implementation field catalogue

The schema keeps extension fields open; this catalogue fixes the v1 meaning of those optional fields. Required fields and enum constraints are enforced by the JSON Schema. All timestamps are RFC 3339 UTC; all binary content is evidence-addressed rather than duplicated in aggregate records.

| Model | Optional v1 fields | Relationships and semantics |
|---|---|---|
| ProjectState | `activeRunId`, `activeArtifactId`, `blockers`, `lastErrorId`, `metadata` | Root aggregate. `hardwareModelId` identifies the graph snapshot; lifecycle enum is in schema. |
| BoardDescriptor | `usb`, `serial`, `vendor`, `architecture`, `voltageV`, `maxGpioCurrentMa` | Discovery adapter produces it. Pin IDs are owned by this descriptor. |
| ComponentInstance | `label`, `firmwareBinding`, `electrical`, `properties`, `evidenceIds` | `driverId` resolves in registry; its properties are assertions with separate provenance. |
| PinDefinition | `number`, `voltageV`, `maxCurrentMa`, `reserved`, `aliases` | Referenced by `Connection`; capability values are the enum in schema. |
| Connection | `signal`, `mode`, `voltageV`, `notes`, `evidenceIds` | `from` and `to` are endpoint IDs (`component:terminal`, `board:pin`). |
| FirmwareArtifact | `sourceDigest`, `toolchainVersion`, `compileProfile`, `path`, `logEvidenceIds` | Immutable compile output; flash/run records point at its ID and digest. |
| HardwareContext | `boardId`, `deviceFingerprint`, `components`, `connections`, `resourceReportId`, `inferences` | Read model joining project graph, discovery snapshot, observations and latest analysis. |
| DiagnosticReport | `window`, `ruleSetVersion`, `generatedAt`, `summary` | References findings and the run/context that supplied its observation window. |
| DiagnosticFinding | `severity`, `recommendations`, `parameters`, `firstSeenAt`, `lastSeenAt` | `evidenceIds` are mandatory citations; confidence ranges from 0 to 1. |
| SerialObservation | `port`, `baudRate`, `decodedText`, `captureId`, `sequence` | Raw bytes are authoritative. Parsed values are derived evidence, not replacements. |
| UsbObservation | `vid`, `pid`, `port`, `serialNumber`, `location`, `correlationId` | Fingerprint permits re-enumeration correlation across port changes. |
| DriverDefinition | `extends`, `electrical`, `wiring`, `pinConstraints`, `knownIssues`, `dataParsing`, `firmwareBindings` | Version-pinned by ComponentInstance and ExperimentDefinition. |
| ExperimentDefinition | `variables`, `requires`, `cleanup`, `tags`, `description` | Step IDs form a deterministic graph. `branch` targets are step IDs only. |
| ExperimentStep | `dependsOn`, `inputs`, `expression`, `onPass`, `onFail`, `evidence`, `retry` | Exactly the fields applicable to its `type` are validated by the step-specific implementation schema. |
| ExperimentRun | `cursor`, `target`, `variables`, `startedAt`, `finishedAt`, `artifactId`, `errorId` | Holds a frozen definition version/context revision and references all events/evidence. |
| HumanActionRequest | `template`, `instructionsKey`, `inputSchema`, `riskAcknowledgements`, `createdAt` | One pending request per run. `token` is opaque, single-use and never logged. |
| HumanActionResponse | `actor`, `respondedAt`, `evidenceIds`, `acknowledgements` | Valid only for the matching unexpired request token; response follows request schema. |
| VerificationEvidence | `mimeType`, `capturedAt`, `observationIds`, `metadata`, `retentionClass` | Content digest addresses immutable stored payload; source enum specifies provenance channel. |
| VerificationReport | `scope`, `checks`, `findings`, `generatedAt`, `comparison` | Verdict enum is passed/failed/inconclusive; physical and simulated evidence remain separated. |
| ResourceReport | `boardId`, `railBudget`, `pinAssignments`, `warnings`, `blockers`, `generatedAt` | Safety gate consumes it; each item names its source constraint and affected resource. |
| SimulationState | `modelVersion`, `clockMs`, `variables`, `observations`, `limitations` | Must identify synthetic observations and assumptions; never shares physical evidence IDs. |
| RuntimeEvent | `timestamp`, `actor`, `correlationId`, `causationId`, `aggregateType`, `aggregateId` | Globally ordered per project by sequence; durable audit source for recovery. |
| ErrorEnvelope | `details`, `operationId`, `correlationId`, `causeCode`, `timestamp` | Category enum is canonical. Underlying error is redacted/structured, never a raw secret-bearing command. |

### Full context example

```json
{
  "apiVersion": "har/v1",
  "kind": "HardwareContext",
  "id": "ctx_01J",
  "revision": 7,
  "projectId": "project_carduino",
  "modelRevision": 4,
  "boardId": "board_uno_1",
  "deviceFingerprint": "usb:2341:0043:SERIAL123",
  "components": ["component_hcsr04_1"],
  "connections": ["connection_trig_d7", "connection_echo_d8"],
  "observations": ["usb_101", "serial_102"],
  "resourceReportId": "resource_4"
}
```
