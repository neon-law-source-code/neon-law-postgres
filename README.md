# PostgreSQL compatibility image

Synthetic compatibility image for PostgreSQL 16.15 (Debian Bookworm), Apache AGE 1.6.0 for PG16,
pgvector 0.8.2, bundled citext 1.6 and pg_trgm 1.6, and built-in English full-text search.
DBOS Rust 0.5.0 is compiled into the test application with Axum; it is not a database extension.

The Containerfile pins the base OCI index and verifies AGE SHA512 and pgvector SHA256.
Application migrations and extension enablement belong to the consuming application, never image initialization.
AGE needs `LOAD 'age'` and `SET search_path = ag_catalog, public` per connection; no preload is assumed.
Use a pool connection initializer in applications. Vector builds disable host-specific CPU optimization.

CI builds and tests AMD64 and ARM64 before publishing a multi-platform index to
`ghcr.io/neon-law-source-code/postgres-age`. It records the published digest in the job summary.
Consumers must pin that digest. Tags alone are not a release contract.

The smoke test proves extension loading and queries, GIN/HNSW index construction, database persistence,
DBOS registration/launch, completed checkpoint replay, and recovery after SIGKILL in the
business-write/checkpoint window. A unique application effect key prevents duplicate writes on replay.
A non-idempotent counter separately demonstrates that the uncheckpointed step actually executes twice.
DBOS runs in the same Tokio process as Axum. No Conductor is required by this test.

This is not a Navigator runtime adapter. Signals, actor attribution, ephemeral workflows, queue limits,
rolling versions, graceful drain, backup/PITR, extension upgrades, tenancy and API parity remain consumer gates.
Rust transactional steps are unavailable. Business writes must commit a unique replay key and their effects
in the same application transaction; external effects require provider idempotency or an outbox with reconciliation.
Never infer exactly-once external effects from successful DBOS checkpoints.

PostgreSQL runs through its official entrypoint, which drops to the postgres account. Kubernetes consumers
must validate volume ownership and operator compatibility before promotion. Rebuild and rerun the full tuple
for every base, extension or SDK upgrade. Pin application versions explicitly for compatible recovery.

Upstream references: https://age.apache.org/download/,
https://github.com/pgvector/pgvector, https://github.com/dbos-inc/dbos-transact-rust.

Bundled AGE and pgvector license files and AGE NOTICE are retained under `/usr/share/doc` in the image.

## Local smoke lane

```bash
docker build -f Containerfile -t postgres-age:smoke .
bash tests/run.sh postgres-age:smoke
```

The test creates and removes only its randomly named PostgreSQL container and volume.
It requires Docker, Cargo with Rust 1.98, curl, and an available port 8080 for synthetic Axum readiness.
For a released artifact, pass `ghcr.io/neon-law-source-code/postgres-age@sha256:<published-index>`
as the test argument. Native CI runners test each architecture; emulated local runs supplement that gate.
