# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Build Milvus binary (requires C++ deps built first)
make milvus

# Build with custom Tantivy features (e.g., Lindera Korean dictionary)
make milvus TANTIVY_FEATURES=lindera-ko-dic

# GPU build
make milvus-gpu

# Build only C++ components
make build-cpp

# Build inside Docker container (consistent environment)
./build/builder.sh make milvus

# Install all build dependencies (platform-specific)
./scripts/install_deps.sh

# Install Go tool dependencies (golangci-lint, mockery, protoc-gen-go, etc.)
make getdeps
```

**Prerequisites**: Go 1.24.12+, CMake 3.26+, gcc 7.5+ or llvm 15+ (macOS), Conan 1.64.1 (NOT 2.x)

**Running the binary** requires setting library paths:
```bash
LD_LIBRARY_PATH=./internal/core/output/lib:lib:$LD_LIBRARY_PATH ./bin/milvus
```

## Testing

```bash
# Start test dependencies (etcd, MinIO, Pulsar)
cd deployments/docker/dev && docker compose up -d

# Run all unit tests (C++ and Go)
make unittest

# Run Go unit tests only
make test-go

# Run tests for a specific component
make test-proxy
make test-querynode
make test-datanode
make test-datacoord
make test-querycoord
make test-rootcoord
make test-indexnode
make test-storage

# Run a single Go test function
source scripts/setenv.sh
go test -v -tags dynamic,test -count=1 ./internal/proxy/ -run TestSearchTask

# Integration tests
make integration-test

# Coverage
make codecov-go    # generates go_coverage.html
```

Go test flags used in CI: `-race -cover -tags dynamic,test -failfast -count=1`

## Linting and Formatting

```bash
# Auto-fix formatting (gofumpt + gci + golangci-lint --fix)
make lint-fix

# Check formatting only (no auto-fix)
make fmt

# Static analysis (golangci-lint)
make static-check

# Full pre-submission verification (build C++, cppcheck, fmt, static-check)
make verifiers
```

Config: `.golangci.yml` — uses gofumpt, gci (import ordering: std → default → milvus-io), revive, gosec, gocritic, etc.

## Architecture

Milvus is a distributed vector database with a component-based architecture. All components compile into a single binary (`./bin/milvus`) and are selected at runtime.

### Components

**Coordinators** (control plane):
- **RootCoord** — global metadata (databases, collections, partitions), DDL operations
- **DataCoord** — segment lifecycle, allocation, flushing, compaction, channel management
- **QueryCoord** (v2) — query load balancing, segment loading/release, resource groups
- **StreamingCoord** — streaming data pipeline management

**Worker nodes** (data plane):
- **Proxy** — client-facing entry point, request routing, authentication, rate limiting
- **DataNode** — data ingestion, segment writes, binlog production
- **QueryNode** (v2) — search/query execution on loaded segments
- **IndexNode** — vector index building

### Communication

- **gRPC**: Request/response between components. Proto definitions in `/pkg/proto/`. Service wrappers in `/internal/distributed/<component>/service.go`.
- **Message Queue**: Event streaming for loose coupling. Supports Pulsar (cluster default), Kafka, RocksMQ (standalone default), NatsMQ. Abstracted in `/pkg/mq/msgstream/`.
- **Metadata**: Stored in etcd/TiKV via KV abstraction (`/internal/kv/`).

### Code Organization

| Directory | Purpose |
|-----------|---------|
| `cmd/` | Entry points and component bootstrap (`roles/roles.go` orchestrates startup) |
| `internal/<component>/` | Core implementation for each component (rootcoord, datacoord, querycoordv2, querynodev2, proxy, datanode, indexnode) |
| `internal/distributed/` | gRPC server/client wrappers for each component |
| `internal/storage/` | Storage abstraction (local, S3, Azure, Aliyun), binlog codec |
| `internal/metastore/` | Metadata catalog interface and KV-backed implementations |
| `internal/types/` | Core interfaces (Component, DataCoordClient, RootCoordClient, etc.) |
| `pkg/proto/` | Protobuf definitions for all services |
| `pkg/mq/` | Message queue abstraction and implementations |
| `pkg/config/` | Parameter management |
| `pkg/util/` | Shared utilities, session management |
| `client/` | Official Go SDK |
| `tests/integration/` | Go integration tests |
| `tests/python_client/` | Python E2E tests |

### Request Flow (Insert example)

Client → **Proxy** (gRPC) → publishes to MsgStream insertion channel → **DataNode** (subscribes) buffers and flushes segments to storage → **QueryNode** loads flushed segments for search. Coordinators monitor via time-tick channels for consistency.

### Key Patterns

- **Task-based DDL**: RootCoord uses task objects (`internal/rootcoord/*_task.go`) as state machines for DDL operations.
- **Dependency Factory**: `internal/util/dependency/factory.go` provides ChunkManager and MsgStreamFactory to all components.
- **Channel-based coordination**: Insertion channels, query channels, statistics channels, and time-tick channels synchronize distributed state.
