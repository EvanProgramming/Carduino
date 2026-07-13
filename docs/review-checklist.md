# Architecture review checklist

This checklist is a hand-off gate before production implementation begins.

- [x] System boundary, responsibility ownership, and explicit non-goals
- [x] Acyclic core/adapters architecture and a single command coordinator
- [x] Purpose, input/output, owned state, interface, dependency, failure, and prohibition for every requested runtime module
- [x] Versioned schema and implementer-facing catalogue for all requested models
- [x] Separate user-reported, observed, inferred, and verified state
- [x] Project, experiment, and device connection state machines with rejected invalid transitions
- [x] Compile/flash, diagnostics, human pause/resume, simulation comparison, and CI flows
- [x] Arduino CLI as an adapter and PlatformIO extension route
- [x] Portable HC-SR04 driver, deterministic DSL, and three experiment examples
- [x] Rule/evidence/confidence diagnostics extension design
- [x] MCP surface, idempotency/mutation/pause/error behavior
- [x] SQLite recovery, safety gate, unified error envelope, and hardware-aware tests
- [x] Repository layout and eight required architectural decisions

Open implementation decisions are deliberately deferred: exact TypeScript package manager, serial library, SQLite binding, adapter process protocol, and first physical test rig inventory. They do not alter the published contracts or boundaries.
