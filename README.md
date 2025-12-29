# gRPC Learning Path 🚀

A structured learning path for **Node.js developers** learning gRPC with **Golang**.

## Why gRPC?

| REST (What you know) | gRPC (What you'll learn) |
|---------------------|--------------------------|
| JSON (text, slow) | Protobuf (binary, 7-10x faster) |
| No type safety | Strong type safety |
| Manual API docs | Auto-generated from `.proto` |
| WebSocket for streaming | Built-in streaming |

---

## Learning Phases

| Phase | Topic | Time |
|-------|-------|------|
| [Phase 1](./phase-1-fundamentals/README.md) | gRPC Fundamentals & First Service | 2-3 hours |
| [Phase 2](./phase-2-patterns/README.md) | 4 gRPC Patterns (Streaming) | 3-4 hours |
| [Phase 3](./phase-3-rest-migration/README.md) | Converting REST to gRPC | 2-3 hours |
| [Phase 4](./phase-4-production/README.md) | Production: Interceptors, Auth, TLS | 3-4 hours |

---

## Prerequisites

```bash
# 1. Install Protocol Buffers compiler
brew install protobuf

# 2. Install Go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 3. Add to PATH (add to ~/.zshrc)
export PATH="$PATH:$(go env GOPATH)/bin"
```

---

## How to Use

1. **Read the README.md** of each phase (theory)
2. **Complete the tasks** in order
3. **Run the code** to verify it works
4. **Move to next phase** after completing all tasks

Start with → [Phase 1: Fundamentals](./phase-1-fundamentals/README.md)
# Grpc
