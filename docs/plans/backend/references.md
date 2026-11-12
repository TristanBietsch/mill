# References

Status: review (2026-11-12 refinement, awaiting the user). The books the backend plan follows, the short key each citation uses, where the plan uses each one and why it fits mill, and where mill departs from the book on purpose.

## How citations work

- Every design decision in the backend plan names its source inline as `[KEY ch. N]` (a chapter) or `[KEY, Pattern]` (a named pattern or concept). Example: `[DDIA ch. 11]`, `[RI ch. 5, Bulkheads]`, `[EIP, Claim Check]`.
- A chapter number is given only where it is certain for the edition listed below. Otherwise the citation names the book and the concept only.
- Chapter numbers refer to the edition in the table. Newer editions may renumber; the concept names stay valid.
- The books justify a choice; they don't override the spec. When the spec and a book disagree, the spec wins and the departure is listed below.

## Bibliography

| Key | Book | Authors | Edition, publisher, year |
|---|---|---|---|
| DDIA | *Designing Data-Intensive Applications* | Martin Kleppmann | 1st ed., O'Reilly, 2017 |
| DBI | *Database Internals* | Alex Petrov | 1st ed., O'Reilly, 2019 |
| FSA | *Fundamentals of Software Architecture* | Mark Richards, Neal Ford | 1st ed., O'Reilly, 2020 |
| BEA | *Building Evolutionary Architectures* | Neal Ford, Rebecca Parsons, Patrick Kua, Pramod Sadalage | 2nd ed., O'Reilly, 2022 |
| SAHP | *Software Architecture: The Hard Parts* | Neal Ford, Mark Richards, Pramod Sadalage, Zhamak Dehghani | 1st ed., O'Reilly, 2021 |
| SRE | *Site Reliability Engineering* | Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy (eds.) | 1st ed., O'Reilly, 2016 |
| SRW | *The Site Reliability Workbook* | Betsy Beyer, Niall Richard Murphy, David K. Rensin, Kent Kawahara, Stephen Thorne (eds.) | 1st ed., O'Reilly, 2018 |
| BSRS | *Building Secure and Reliable Systems* | Heather Adkins, Betsy Beyer, Paul Blankinship, Piotr Lewandowski, Ana Oprea, Adam Stubblefield | 1st ed., O'Reilly, 2020 |
| OE | *Observability Engineering* | Charity Majors, Liz Fong-Jones, George Miranda | 1st ed., O'Reilly, 2022 |
| SEG | *Software Engineering at Google* | Titus Winters, Tom Manshreck, Hyrum Wright (eds.) | 1st ed., O'Reilly, 2020 |
| CIG | *Concurrency in Go* | Katherine Cox-Buday | 1st ed., O'Reilly, 2017 |
| LG | *Learning Go* | Jon Bodner | 2nd ed., O'Reilly, 2024 |
| BM | *Building Microservices* | Sam Newman | 2nd ed., O'Reilly, 2021 |
| RI | *Release It!* | Michael T. Nygard | 2nd ed., Pragmatic Bookshelf, 2018 |
| EIP | *Enterprise Integration Patterns* | Gregor Hohpe, Bobby Woolf | 1st ed., Addison-Wesley, 2003 |
| TAOUP | *The Art of UNIX Programming* | Eric S. Raymond | 1st ed., Addison-Wesley, 2003 |
| CA | *Clean Architecture* | Robert C. Martin | 1st ed., Prentice Hall, 2017 |
| DDD | *Domain-Driven Design* | Eric Evans | 1st ed., Addison-Wesley, 2003 |
| SP | *Systems Performance* | Brendan Gregg | 2nd ed., Addison-Wesley, 2020 |

The first thirteen are O'Reilly titles. The last six are not, but each is the standard reference for its topic: stability patterns (RI), messaging and pipes-and-filters vocabulary (EIP), the Unix filter tradition mill copies at the shell (TAOUP), the dependency rule (CA), bounded contexts and aggregates (DDD), and performance method (SP).

## Chapters cited

Only these chapter numbers are used in the plan. Anything else is cited by concept.

| Key | Chapters cited |
|---|---|
| DDIA | 3 Storage and Retrieval · 4 Encoding and Evolution · 7 Transactions · 8 The Trouble with Distributed Systems · 10 Batch Processing · 11 Stream Processing · 12 The Future of Data Systems |
| DBI | 3 File Formats · 5 Transaction Processing and Recovery · 7 Log-Structured Storage · 9 Failure Detection |
| FSA | 3 Modularity · 6 Measuring and Governing Architecture Characteristics · 10 Layered Architecture Style · 11 Pipeline Architecture Style · 12 Microkernel Architecture Style · 19 Architecture Decisions |
| BEA | 2 Fitness Functions · 4 Automating Architectural Governance · 6 Evolutionary Data |
| SAHP | 8 Reuse Patterns · 13 Contracts |
| SRE | 3 Embracing Risk · 4 Service Level Objectives · 6 Monitoring Distributed Systems · 21 Handling Overload · 22 Addressing Cascading Failures · 24 Distributed Periodic Scheduling with Cron · 25 Data Processing Pipelines · 26 Data Integrity |
| SRW | 2 Implementing SLOs · 4 Monitoring · 13 Data Processing Pipelines · 14 Configuration Design and Best Practices · 15 Configuration Specifics |
| BSRS | 2 Understanding Adversaries · 5 Design for Least Privilege · 6 Design for Understandability · 8 Design for Resilience · 9 Design for Recovery · 12 Writing Code · 13 Testing Code · 15 Investigating Systems |
| OE | 5 Structured Events Are the Building Blocks of Observability · 6 Stitching Events into Traces · 7 Instrumentation with OpenTelemetry · 12 Using Service-Level Objectives for Reliability |
| SEG | 1 What Is Software Engineering? · 11 Testing Overview · 12 Unit Testing · 13 Test Doubles · 14 Larger Testing · 15 Deprecation · 20 Static Analysis · 21 Dependency Management · 23 Continuous Integration |
| CIG | 2 Modeling Your Code: Communicating Sequential Processes · 3 Go's Concurrency Building Blocks · 4 Concurrency Patterns in Go · 5 Concurrency at Scale |
| LG | 7 Types, Methods, and Interfaces · 9 Errors · 10 Modules, Packages, and Imports · 12 Concurrency in Go · 14 The Context · 15 Writing Tests |
| BM | 12 Resiliency |
| RI | 4 Stability Antipatterns · 5 Stability Patterns · 10 Control Plane · 13 Design for Deployment · 14 Handling Versions · 17 Chaos Engineering |
| TAOUP | 1 Philosophy · 5 Textuality · 7 Multiprogramming · 10 Configuration · 11 Interfaces |
| CA | 11 DIP: The Dependency Inversion Principle · 14 Component Coupling · 22 The Clean Architecture · 26 The Main Component |
| DDD | 4 Isolating the Domain · 6 The Life Cycle of a Domain Object · 14 Maintaining Model Integrity |
| EIP, SP | patterns and methods by name only |

## Per book: where it's used, why, and where mill departs

### DDIA

- **Used in:** [layers.md](layers.md) (L1, L3), [storage.md](storage.md), [engine.md](engine.md) §3, §10, [data-lifecycle.md](data-lifecycle.md), [versioning.md](versioning.md), [concurrency.md](concurrency.md).
- **Why it fits:** mill's run store is an event log with derived views (`run.json`, SQLite index), which is the "log as source of truth, everything else derived" design of ch. 11 and ch. 12. Ch. 4's forward and backward compatibility rules are principle 12 (additive evolution) in other words. Ch. 3 explains why an append-only log plus a rebuildable index is cheap and crash-friendly. Ch. 7's compare-and-set is the lost-update guard on pipeline saves (`base_hash`). Ch. 8 is why durations use the monotonic clock and why `max_wall` counts active time. Ch. 10 frames steps as Unix-style batch filters.
- **Departs:** no log compaction or snapshotting of a run's event log. A run is bounded (thousands of events, not millions), so replay from the start stays under the B3 budget and compaction would add a second source of truth. No replication: one machine, one writer.

### DBI

- **Used in:** [storage.md](storage.md) §8, [data-lifecycle.md](data-lifecycle.md), [versioning.md](versioning.md), [resilience.md](resilience.md) (part health).
- **Why it fits:** ch. 5 is the write-ahead rule mill applies to processes (`NodeRunStarting` is written and fsynced before spawning) and the redo-style recovery of `Engine.Recover`. Ch. 3 covers versioned file formats and torn-write detection, used for the `log_v` header and torn-tail truncation. Ch. 7 is the append-only-plus-merge shape of the CAS and GC. Ch. 9 (heartbeats, timeouts as failure detectors) is the part `Ping`/`Pong` rule.
- **Departs:** mill writes no page cache, B-tree or checksummed block format of its own; it relies on SQLite for the index and on line-oriented JSON plus fsync for the log. Per-record checksums are not used; a torn last line is detected by JSON validity and the missing newline.

### FSA

- **Used in:** [layers.md](layers.md) (whole file), [architecture.md](architecture.md), [00-overview.md](00-overview.md).
- **Why it fits:** ch. 10 defines layers of isolation, closed versus open layers and the architecture sinkhole antipattern, which shape the layer rules. Ch. 11 (pipeline style: pipes and filters) is mill's user-facing model. Ch. 12 (microkernel: a small core plus plug-ins) is principle 7, "everything is a part": the engine is the core, parts are plug-ins. Ch. 3 (cohesion, coupling, connascence) decides package boundaries. Ch. 6 introduces fitness functions to govern characteristics. Ch. 19 is the ADR practice `docs/agents/DECISIONS.md` follows.
- **Departs:** the book treats layered architecture as a monolithic, technically partitioned style that scales poorly. mill is a single local binary pair, which is where the style fits. mill's layers are **open (relaxed)** by default (a layer may import any lower layer) and closed only at named boundaries (the surface), where the book's default is closed layers.

### BEA

- **Used in:** [layers.md](layers.md) §8 (fitness functions), [milestones.md](milestones.md) (per-milestone fitness functions), [testing.md](testing.md), [data-lifecycle.md](data-lifecycle.md) (migration = rebuild).
- **Why it fits:** ch. 2 and ch. 4 turn "lower layers never import higher ones" from a convention into an automated, failing check in CI, and give the ratchet idea used for exceptions (the list may only shrink). Ch. 6 (evolutionary data) backs rebuild-not-migrate for the derived index.
- **Departs:** none of substance. mill uses Go's own tooling (`go list`, golangci-lint) instead of a dedicated architecture-test framework.

### SAHP

- **Used in:** [tech-stack.md](tech-stack.md) §6 (copy from rex), [versioning.md](versioning.md), [protocols.md](protocols.md).
- **Why it fits:** ch. 8 lists code replication versus shared library versus shared service and their trade-offs; mill's copy-and-diverge from rex (ADR-026) is code replication chosen on purpose while the two codebases move fast. Ch. 13 (strict versus loose contracts, consumer-driven contracts) shapes the wire and parts protocols: loose (unknown fields ignored), with a conformance suite as the consumer-side contract test.
- **Departs:** the book is about distributed systems; mill's "services" are processes on one machine. Only the reuse and contract chapters apply.

### SRE

- **Used in:** [observability.md](observability.md), [resilience.md](resilience.md), [data-lifecycle.md](data-lifecycle.md), [operations.md](operations.md), trigger sections of [engine.md](engine.md) and [architecture.md](architecture.md).
- **Why it fits:** ch. 6 gives the four golden signals used for `milld`. Ch. 4 and ch. 3 give the vocabulary for targets. Ch. 21 and ch. 22 (overload, cascading failures) justify bounded queues, load shedding and dropping slow clients. Ch. 24 (cron) is the reference for `catch_up`, skipping versus queueing missed fires, and why cron jobs should be idempotent. Ch. 25 covers pipeline health and stuck work. Ch. 26 (data integrity) frames retention, GC grace periods and restore-first backups.
- **Departs:** mill is a single-user local tool. There are no SLAs, no error-budget policy that freezes releases, no paging. Targets are engineering budgets checked by benchmarks and `mill doctor`, not promises.

### SRW

- **Used in:** [observability.md](observability.md), [configuration.md](configuration.md).
- **Why it fits:** ch. 2 and ch. 4 are the practical way to pick SLIs and targets; ch. 5's alerting advice is mapped onto the board's "failed" header instead of pages. Ch. 14 and ch. 15 (configuration design: separate philosophy from mechanics, validate early, keep config reviewable, safe reload) shape the config layering and reload.
- **Departs:** no burn-rate alerting; a local tool shows state, it doesn't page.

### BSRS

- **Used in:** [security.md](security.md) (threat model by layer), [errors.md](errors.md), [operations.md](operations.md), [resilience.md](resilience.md).
- **Why it fits:** ch. 2 (adversaries) frames the threat model. Ch. 5 (least privilege) is the secret grant per node-run, the env allowlist and the trust gate for cloned projects. Ch. 6 (understandability) motivates small invariant-bearing layers and one teardown path. Ch. 8 and ch. 9 (resilience, recovery) are "crash, don't guess" and recovery from the log. Ch. 12 (writing code: hardened types that make misuse hard) is the opaque `secrets.Value` and the CEL secret handle. Ch. 13 (fuzzing, testing) backs the fuzz targets. Ch. 15 (investigating systems, logging) shapes logs and the support bundle.
- **Departs:** no multi-party authorization or production access controls; mill runs as one user on that user's machine.

### OE

- **Used in:** [observability.md](observability.md).
- **Why it fits:** ch. 5 argues for wide structured events as the base unit; mill's event log already is one, so traces are derived from it (ch. 6) instead of instrumented twice. Ch. 7 (OpenTelemetry) is the format of the optional `otel` sink. Ch. 12 (SLOs) informs the local targets.
- **Departs:** the book sends high-cardinality events to a hosted backend. mill keeps them local in files and exports only on request (principle 15, local-first). Metrics use low-cardinality labels because they live in a small in-process registry.

### SEG

- **Used in:** [testing.md](testing.md), [layers.md](layers.md) §7, [versioning.md](versioning.md), [tech-stack.md](tech-stack.md).
- **Why it fits:** ch. 11 to ch. 14 give the test size model (unit, component, larger tests), the preference for fakes over mocks (ch. 13, the echo agent and fake CLIs), and hermetic tests. Ch. 20 (static analysis in the workflow) is golangci-lint plus the layer checker. Ch. 21 (dependency management) backs pinned versions and few dependencies. Ch. 15 (deprecation) is the policy for retiring settings, fields and codes. Ch. 1 names Hyrum's Law, the reason JSON outputs and error codes are treated as contracts.
- **Departs:** monorepo and Bazel practices don't apply; mill is one Go module built with `go`.

### CIG

- **Used in:** [concurrency.md](concurrency.md), [architecture.md](architecture.md) §5, [resilience.md](resilience.md).
- **Why it fits:** ch. 2 (CSP) is the run-actor and scheduler-loop model: share memory by communicating. Ch. 3 covers channels, `select` and `sync` primitives. Ch. 4 gives confinement (one writer per run), preventing goroutine leaks (every goroutine has an owner that can stop it), pipelines, queuing and the `context` package. Ch. 5 gives error propagation across goroutines, timeouts and cancellation, heartbeats (stall detection), rate limiting and healing unhealthy goroutines (part restarts with backoff).
- **Departs:** the book leans on channels for most coordination; mill also uses copy-on-write snapshots behind a mutex for read-mostly state (settings, registry, subscriber list), which is simpler and cheaper there.

### LG

- **Used in:** [layers.md](layers.md), [errors.md](errors.md), [concurrency.md](concurrency.md), [testing.md](testing.md).
- **Why it fits:** ch. 7 (accept interfaces, return structs; small interfaces declared by the consumer) is how ports are placed in the lower layer. Ch. 9 (wrapping with `%w`, `errors.Is` and `errors.As`, sentinel versus typed errors) is the error-wrapping rule. Ch. 10 (`internal/` packages, import cycles) underpins the package map. Ch. 12 and ch. 14 (goroutines, `context` cancellation and values) back the context rules. Ch. 15 covers table tests, fuzzing and benchmarks.
- **Departs:** none.

### BM

- **Used in:** [resilience.md](resilience.md).
- **Why it fits:** ch. 12 collects timeouts, retries, bulkheads, circuit breakers and idempotency in one place, applied to mill's integration points (agent CLIs, HTTP, MCP servers, parts).
- **Departs:** mill is not a microservice system. Nothing else from the book applies.

### RI

- **Used in:** [resilience.md](resilience.md) (whole file), [engine.md](engine.md) §5–§7, §9, [operations.md](operations.md), [versioning.md](versioning.md).
- **Why it fits:** ch. 4 lists the antipatterns mill must not have (integration points without timeouts, blocked threads, unbounded result sets, cascading failures). Ch. 5 is the pattern catalog mill uses by name: Timeouts, Circuit Breaker, Bulkheads (slot pools), Steady State (retention and GC), Fail Fast (secrets and lint before any work), Let It Crash ("crash, don't guess"), Handshaking (`Hello`, `Describe`), Test Harnesses (fake CLIs), Create Back Pressure, Shed Load, Governor (budgets). Ch. 10 (control plane) frames `mill doctor` and the daemon controls. Ch. 13 and ch. 14 frame drain-and-restart upgrades and protocol versioning. Ch. 17 backs crash-point exploration.
- **Departs:** Circuit Breaker in the book fails callers fast. mill holds work in the queue while a breaker is open instead of failing it, so a provider outage doesn't burn retry attempts (decision D36 in [open-decisions.md](open-decisions.md)).

### EIP

- **Used in:** [engine.md](engine.md) §2–§3, [protocols.md](protocols.md), [storage.md](storage.md), [architecture.md](architecture.md), [layers.md](layers.md).
- **Why it fits:** the vocabulary for mill's data plane: Pipes and Filters (nodes and edges), Message and Envelope Wrapper (`core.Item`, the JSONL envelope), Correlation Identifier (`id`), Format Indicator (`v`), Splitter and Aggregator (`each` mode, `merge`), Content-Based Router (`if`, `switch`), Dead Letter Channel (`on_error: route` to the `error` port), Claim Check (large payloads as CAS artifacts referenced by hash), Idempotent Receiver (webhook and poll dedupe), Message Store (the event log), Wire Tap (sinks), Durable Subscriber (`Subscribe` from a seq), Competing Consumers (none: one scheduler grants work).
- **Departs:** no broker. Channels are in-process Go channels and files; guaranteed delivery comes from fsync before commit, not from a messaging system.

### TAOUP

- **Used in:** [layers.md](layers.md), [engine.md](engine.md) §3, [configuration.md](configuration.md), [errors.md](errors.md), [observability.md](observability.md).
- **Why it fits:** ch. 1 (Rules of Modularity, Composition, Separation, Silence, Repair) and ch. 11 (the filter interface pattern) are principle 6, "pipes all the way down", and the stdout-results/stderr-progress split. Ch. 5 (textuality) is principle 16, boring formats. Ch. 7 (multiprogramming: cooperating processes over pipes) backs parts as processes. Ch. 10 (configuration: where config lives, precedence of system, user, environment and flags) shapes config layers. The Rule of Repair ("fail noisily and as soon as possible") backs fail-fast compile and secret resolution.
- **Departs:** the Rule of Silence would print nothing on success; mill prints progress to stderr when stderr is a terminal and stays silent otherwise, so pipes and cron logs stay clean.

### CA

- **Used in:** [layers.md](layers.md) (dependency rule, ports), [architecture.md](architecture.md) §3.
- **Why it fits:** ch. 22's dependency rule (source dependencies point inward, toward policy) is the layer rule. Ch. 11 (dependency inversion) places interfaces such as `engine.PartSource` in the lower layer and implementations above, injected at the top. Ch. 14 (acyclic dependencies, stable dependencies) backs keeping `core` and the contracts small and stable. Ch. 26 (the Main component as the dirtiest, outermost part) is `daemon/boot` and `surface/cli/local`, the only places that wire concrete types together.
- **Departs:** the book would wrap frameworks behind interfaces. mill lets L2 use cel-go and yaml/v4 directly: they are the language, and an abstraction over them would add indirection with no second implementation in sight.

### DDD

- **Used in:** [layers.md](layers.md), [engine.md](engine.md), [execution.md](execution.md) §5.
- **Why it fits:** ch. 4 introduces layered architecture to isolate the domain; L1 is that isolated domain (`core`, the event catalog, the reducer). Ch. 6 (aggregates with one root guarding invariants) is the run actor: one writer per run, every change through it. Ch. 14 (bounded contexts, published language, anticorruption layer) maps to: the parts protocol and wire protocol as published languages, and the agent registry's error normalization as an anticorruption layer between agent CLIs and mill's error classes.
- **Departs:** no repositories or factories as separate abstractions beyond `store.RunStore`; the event log is the repository.

### SP

- **Used in:** [performance.md](performance.md).
- **Why it fits:** the USE method (utilization, saturation, errors per resource) structures the capacity checks for CPU, memory, disk, file descriptors and goroutines, and complements the golden signals for the daemon's own resources.
- **Departs:** mill doesn't profile the OS for the user; it applies the method to its own resources only.
