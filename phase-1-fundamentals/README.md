# Phase 1: gRPC Fundamentals 🎯

## What You'll Learn
- What is gRPC and why use it
- Protocol Buffers (protobuf) syntax
- Generate Go code from `.proto` files
- Build your first gRPC server and client

---

## 📖 Theory

### What is gRPC?

**gRPC** = **g**oogle **R**emote **P**rocedure **C**all

Think of it like calling a function on another computer:

```
// Node.js REST (what you know)
fetch('/api/users/123')           // HTTP request with JSON

// gRPC (what you'll learn)  
client.GetUser({ id: 123 })       // Like calling a local function
```

### How gRPC Works

```
┌──────────────┐         .proto file          ┌──────────────┐
│    Client    │ ◄──────────────────────────► │    Server    │
│   (Go code)  │       (shared contract)      │   (Go code)  │
└──────────────┘                              └──────────────┘
        │                                            │
        │  1. Client calls GetUser(id: 123)          │
        │ ──────────────────────────────────────────►│
        │                                            │
        │  2. Server returns User{name: "Keshav"}    │
        │ ◄──────────────────────────────────────────│
```

### Protocol Buffers (Protobuf)

Protobuf is the **language** gRPC uses. It's like JSON but:
- **Smaller** (binary, not text)
- **Faster** (no parsing overhead)
- **Type-safe** (compile-time errors)

---

## 📝 Protobuf Syntax

### Basic `.proto` File

```protobuf
syntax = "proto3";                    // Always proto3

package user;                         // Package name

option go_package = "./pb";           // Where Go code goes

// Message = Data structure (like TypeScript interface)
message User {
  int32 id = 1;                       // Field number, not default value!
  string name = 2;
  string email = 3;
  bool is_active = 4;
}

// Request message
message GetUserRequest {
  int32 id = 1;
}

// Response message  
message GetUserResponse {
  User user = 1;
}

// Service = API definition (like Express routes)
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}
```

### Field Types (Protobuf → Go)

| Protobuf | Go | Example |
|----------|-----|---------|
| `int32` | `int32` | `id = 1` |
| `int64` | `int64` | `age = 2` |
| `string` | `string` | `name = 3` |
| `bool` | `bool` | `active = 4` |
| `repeated string` | `[]string` | `tags = 5` |
| `Message` | `*Message` | `user = 6` |

### Field Numbers (Important!)

```protobuf
message User {
  string name = 1;    // 1 is the field NUMBER, not value!
  string email = 2;   // Used for binary encoding
}

// ❌ NEVER reuse or change field numbers after deployment
// ❌ NEVER use 19000-19999 (reserved)
// ✅ Can skip numbers (1, 2, 5 is valid)
```

---

## 💻 Tasks

### Task 1: Setup Project

```bash
cd /Users/keshavsharma/Grpc/phase-1-fundamentals

# Create project structure
mkdir -p user-service/{proto,pb,server,client}
cd user-service

# Initialize Go module
go mod init user-service

# Add gRPC dependencies
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

**Expected structure:**
```
user-service/
├── go.mod
├── proto/
│   └── user.proto      # Your proto definition
├── pb/                  # Generated Go code (auto)
├── server/
│   └── main.go         # gRPC server
└── client/
    └── main.go         # gRPC client
```

---

### Task 2: Create Proto File

Create `proto/user.proto`:

```protobuf
syntax = "proto3";

package user;

option go_package = "./pb";

// User message
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

// GetUser
message GetUserRequest {
  int32 id = 1;
}

message GetUserResponse {
  User user = 1;
}

// CreateUser
message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

// Service definition
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}
```

---

### Task 3: Generate Go Code

```bash
# From user-service directory
protoc --go_out=. --go-grpc_out=. proto/user.proto
```

**What gets generated in `pb/`:**
- `user.pb.go` - Message structs (User, GetUserRequest, etc.)
- `user_grpc.pb.go` - Service interface (UserServiceServer, UserServiceClient)

---

### Task 4: Implement Server

Create `server/main.go`:

```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	pb "user-service/pb"
)

// Server struct implements UserServiceServer interface
type server struct {
	pb.UnimplementedUserServiceServer
}

// In-memory "database"
var users = map[int32]*pb.User{
	1: {Id: 1, Name: "Keshav", Email: "keshav@example.com"},
	2: {Id: 2, Name: "John", Email: "john@example.com"},
}
var nextID int32 = 3

// GetUser implementation
func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
	log.Printf("GetUser called with ID: %d", req.Id)
	
	user, exists := users[req.Id]
	if !exists {
		return nil, fmt.Errorf("user not found: %d", req.Id)
	}
	
	return &pb.GetUserResponse{User: user}, nil
}

// CreateUser implementation
func (s *server) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
	log.Printf("CreateUser called: %s, %s", req.Name, req.Email)
	
	user := &pb.User{
		Id:    nextID,
		Name:  req.Name,
		Email: req.Email,
	}
	users[nextID] = user
	nextID++
	
	return &pb.CreateUserResponse{User: user}, nil
}

func main() {
	// 1. Listen on port
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// 2. Create gRPC server
	grpcServer := grpc.NewServer()

	// 3. Register our service
	pb.RegisterUserServiceServer(grpcServer, &server{})

	log.Println("gRPC Server running on :50051")
	
	// 4. Start serving
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

**Compare to Express.js:**
```javascript
// Express.js
app.get('/users/:id', (req, res) => {
  res.json(users[req.params.id]);
});

// gRPC Go
func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
  return &pb.GetUserResponse{User: users[req.Id]}, nil
}
```

---

### Task 5: Implement Client

Create `client/main.go`:

```go
package main

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "user-service/pb"
)

func main() {
	// 1. Connect to server
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	// 2. Create client
	client := pb.NewUserServiceClient(conn)

	// 3. Set timeout
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	defer cancel()

	// 4. Call GetUser
	log.Println("Calling GetUser(1)...")
	res, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
	if err != nil {
		log.Fatalf("GetUser failed: %v", err)
	}
	log.Printf("User: %+v", res.User)

	// 5. Call CreateUser
	log.Println("Calling CreateUser...")
	createRes, err := client.CreateUser(ctx, &pb.CreateUserRequest{
		Name:  "New User",
		Email: "new@example.com",
	})
	if err != nil {
		log.Fatalf("CreateUser failed: %v", err)
	}
	log.Printf("Created User: %+v", createRes.User)
}
```

---

### Task 6: Run and Test

**Terminal 1 - Start Server:**
```bash
cd /Users/keshavsharma/Grpc/phase-1-fundamentals/user-service
go run server/main.go
# Output: gRPC Server running on :50051
```

**Terminal 2 - Run Client:**
```bash
cd /Users/keshavsharma/Grpc/phase-1-fundamentals/user-service
go run client/main.go
# Output:
# Calling GetUser(1)...
# User: id:1 name:"Keshav" email:"keshav@example.com"
# Calling CreateUser...
# Created User: id:3 name:"New User" email:"new@example.com"
```

---

## ✅ Checklist

Before moving to Phase 2, make sure you can:

- [ ] Explain what gRPC is and why it's faster than REST
- [ ] Write a `.proto` file with messages and services
- [ ] Generate Go code using `protoc`
- [ ] Create a gRPC server that implements the service
- [ ] Create a gRPC client that calls the server
- [ ] Successfully run server and client

---

## 🔗 Next: [Phase 2 - gRPC Patterns](../phase-2-patterns/README.md)
