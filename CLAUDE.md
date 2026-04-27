# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Goal

This is a learning project. The user is a beginner studying CubeFS source code to prepare for contributing at work. All study notes and documentation should be written under `note/`.

## Note Structure

```
note/
├── 00-环境配置/          # Environment setup
├── 01-metanode/          # MetaNode notes
├── 02-master/            # Master notes (start with 00-初学者指南.md)
├── 03-datanode/          # DataNode notes
├── 09-tiglabs-raft/      # Raft library deep dive
│   ├── 00-初学者指南.md   # Beginner guide
│   └── layer1~7          # Layer by layer
└── ...
```

When creating notes:
- Use top-down organization (overview first, then details)
- Include actual source code snippets with file:line references
- Use tables for method listings (not pseudo-interface code)
- Write in Chinese (user's preference)

## Build Commands

```bash
# Build all components (requires Linux/Darwin, uses RocksDB with CGO)
make build

# Build individual components
make server      # Master, MetaNode, DataNode, ObjectNode, AuthNode
make client      # FUSE client
make cli         # Command-line tools
make blobstore   # Erasure coding subsystem
make libsdk      # C library for SDK

# Run tests via Docker (recommended)
./docker/run_docker.sh --test           # All tests
./docker/run_docker.sh --testcubefs     # CubeFS core tests only
./docker/run_docker.sh --testblobstore  # BlobStore tests only

# Code formatting check
./docker/run_docker.sh --format

# Run single package test
go test -v ./metanode/... -run TestXxx
```

## Architecture Overview

CubeFS is a CNCF graduated distributed file and object storage system with separation of metadata and data:

```
┌──────────────────────────────────────────────────────────────────┐
│                         Access Layer                              │
│  client/ (FUSE)    objectnode/ (S3)    sdk/ (libsdk)             │
└────────────────────────────┬─────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────┐
│                      Control Plane                                │
│                       master/                                     │
│         (Cluster management, volume, partition scheduling)       │
└────────────────────────────┬─────────────────────────────────────┘
                             │
┌────────────────────────────┴─────────────────────────────────────┐
│                       Data Plane                                  │
│  metanode/              datanode/              blobstore/         │
│  (Metadata: inode,      (Data: extent-based    (Erasure coding   │
│   dentry, Raft)         replication)           storage)           │
└──────────────────────────────────────────────────────────────────┘
```

**Key subsystems:**

- **master/** - Cluster master managing volumes, partitions, topology. Single Raft group for cluster metadata.
- **metanode/** - Metadata service. Each MetaPartition is a Raft group storing inodes/dentries. Uses RocksDB.
- **datanode/** - Data service. Stores extents with 3-way replication. No Raft (uses primary-backup).
- **blobstore/** - Independent erasure coding subsystem with its own API and components.
- **raftstore/** - CubeFS wrapper around `depends/tiglabs/raft/` Multi-Raft implementation.
- **remotecache/** - FlashNode remote cache layer for hybrid cloud acceleration.
- **proto/** - Shared protocol definitions and API structures.
- **util/** - Common utilities (logging, pool, btree, etc.).

## Raft Implementation

CubeFS uses a vendored Multi-Raft library at `depends/tiglabs/raft/`. Key concepts:

- **RaftServer** (`server.go`) - Manages multiple Raft groups in one process
- **raft** (`raft.go`) - Single Raft group instance with run()/runApply() goroutines
- **raftFsm** (`raft_fsm*.go`) - Protocol state machine (Leader/Follower/Candidate)
- **StateMachine** interface - Implemented by MetaNode/Master for state application

## Code Style

- Use `gofumpt` for formatting: `gofumpt -l -w .`
- Follow Angular commit style: `<type>(<scope>): <subject>`
- Sign commits: `git commit -s`
- Lint with golangci-lint v1.43.0

## Dependencies

- Go 1.18+
- RocksDB (built from `depends/` via build.sh)
- CGO required for client and storage components
- Docker for testing environment
