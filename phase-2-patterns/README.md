# Phase 2: gRPC Patterns 🔄

## What You'll Learn
- All 4 gRPC communication patterns
- When to use each pattern
- Implement streaming in Go

---

## 📖 Theory: The 4 gRPC Patterns

### Overview

| Pattern | Request | Response | Use Case |
|---------|---------|----------|----------|
| **Unary** | 1 | 1 | Normal API (like REST) |
| **Server Streaming** | 1 | Many | Live feeds, logs, large data |
| **Client Streaming** | Many | 1 | File upload, batch insert |
| **Bidirectional** | Many | Many | Chat, real-time sync |

---

### Pattern 1: Unary RPC (Already done in Phase 1)

```
Client ──── Request ────► Server
Client ◄─── Response ──── Server
```

```protobuf
rpc GetUser(GetUserRequest) returns (GetUserResponse);
```

---

### Pattern 2: Server Streaming

Client sends ONE request, Server sends MANY responses.

```
Client ──── Request ────────────────────► Server
Client ◄─── Response 1 ────────────────── Server
Client ◄─── Response 2 ────────────────── Server
Client ◄─── Response 3 ────────────────── Server
Client ◄─── (stream ends) ─────────────── Server
```

**Use Cases:**
- Downloading large files in chunks
- Live log streaming
- Real-time price updates
- Search results as they're found

```protobuf
// Note: stream keyword before response
rpc GetAllUsers(GetAllUsersRequest) returns (stream User);
```

---

### Pattern 3: Client Streaming

Client sends MANY requests, Server sends ONE response.

```
Client ──── Request 1 ────────────────────► Server
Client ──── Request 2 ────────────────────► Server
Client ──── Request 3 ────────────────────► Server
Client ──── (done) ───────────────────────► Server
Client ◄─── Response ─────────────────────── Server
```

**Use Cases:**
- File upload in chunks
- Batch data insertion
- Sending sensor data

```protobuf
// Note: stream keyword before request
rpc UploadUsers(stream User) returns (UploadResponse);
```

---

### Pattern 4: Bidirectional Streaming

Both send MANY messages simultaneously.

```
Client ──── Message 1 ────► ◄──── Message A ──── Server
Client ──── Message 2 ────► ◄──── Message B ──── Server
Client ──── Message 3 ────► ◄──── Message C ──── Server
```

**Use Cases:**
- Chat applications
- Real-time collaboration
- Gaming
- Live dashboards

```protobuf
// Note: stream on BOTH sides
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

---

## 💻 Tasks

### Task 1: Create Chat Service Proto

Create `/Users/keshavsharma/Grpc/phase-2-patterns/chat-service/proto/chat.proto`:

```protobuf
syntax = "proto3";

package chat;

option go_package = "./pb";

message User {
  string id = 1;
  string name = 2;
}

message ChatMessage {
  string from = 1;
  string message = 2;
  int64 timestamp = 3;
}

message Empty {}

// ========== Responses ==========
message GetUsersResponse {
  repeated User users = 1;  // repeated = array
}

message SendMessagesResponse {
  int32 count = 1;
}

// ========== Service ==========
service ChatService {
  // Pattern 1: Unary - Get single user
  rpc GetUser(User) returns (User);
  
  // Pattern 2: Server Streaming - Get all users as stream
  rpc ListUsers(Empty) returns (stream User);
  
  // Pattern 3: Client Streaming - Send many messages, get count
  rpc SendMessages(stream ChatMessage) returns (SendMessagesResponse);
  
  // Pattern 4: Bidirectional - Real-time chat
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

---

### Task 2: Setup and Generate

```bash
cd /Users/keshavsharma/Grpc/phase-2-patterns
mkdir -p chat-service/{proto,pb,server,client}
cd chat-service

go mod init chat-service
go get google.golang.org/grpc
go get google.golang.org/protobuf

# Generate
protoc --go_out=. --go-grpc_out=. proto/chat.proto
```

---

### Task 3: Implement Server

Create `server/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"time"

	"google.golang.org/grpc"
	pb "chat-service/pb"
)

type server struct {
	pb.UnimplementedChatServiceServer
	clients map[string]pb.ChatService_ChatServer
	mu      sync.RWMutex
}

// Pattern 1: Unary
func (s *server) GetUser(ctx context.Context, req *pb.User) (*pb.User, error) {
	log.Printf("GetUser: %s", req.Id)
	return &pb.User{Id: req.Id, Name: "User " + req.Id}, nil
}

// Pattern 2: Server Streaming
func (s *server) ListUsers(req *pb.Empty, stream pb.ChatService_ListUsersServer) error {
	log.Println("ListUsers: streaming users...")
	
	users := []pb.User{
		{Id: "1", Name: "Keshav"},
		{Id: "2", Name: "John"},
		{Id: "3", Name: "Jane"},
	}
	
	for _, user := range users {
		if err := stream.Send(&user); err != nil {
			return err
		}
		time.Sleep(500 * time.Millisecond) // Simulate delay
	}
	
	return nil // Stream ends
}

// Pattern 3: Client Streaming
func (s *server) SendMessages(stream pb.ChatService_SendMessagesServer) error {
	log.Println("SendMessages: receiving stream...")
	
	var count int32 = 0
	
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			// Client done sending
			return stream.SendAndClose(&pb.SendMessagesResponse{Count: count})
		}
		if err != nil {
			return err
		}
		
		log.Printf("Received: %s: %s", msg.From, msg.Message)
		count++
	}
}

// Pattern 4: Bidirectional Streaming
func (s *server) Chat(stream pb.ChatService_ChatServer) error {
	log.Println("Chat: bidirectional stream started...")
	
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		
		log.Printf("[%s]: %s", msg.From, msg.Message)
		
		// Echo back with modification
		reply := &pb.ChatMessage{
			From:      "Server",
			Message:   fmt.Sprintf("Echo: %s", msg.Message),
			Timestamp: time.Now().Unix(),
		}
		
		if err := stream.Send(reply); err != nil {
			return err
		}
	}
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
	s := grpc.NewServer()
	pb.RegisterChatServiceServer(s, &server{
		clients: make(map[string]pb.ChatService_ChatServer),
	})
	
	log.Println("Chat server running on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

---

### Task 4: Implement Client

Create `client/main.go`:

```go
package main

import (
	"context"
	"io"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "chat-service/pb"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewChatServiceClient(conn)
	ctx := context.Background()

	// Test all 4 patterns
	testUnary(ctx, client)
	testServerStreaming(ctx, client)
	testClientStreaming(ctx, client)
	testBidirectional(ctx, client)
}

// Pattern 1: Unary
func testUnary(ctx context.Context, client pb.ChatServiceClient) {
	log.Println("\n=== Pattern 1: Unary ===")
	
	res, err := client.GetUser(ctx, &pb.User{Id: "123"})
	if err != nil {
		log.Fatalf("GetUser failed: %v", err)
	}
	log.Printf("Got user: %+v", res)
}

// Pattern 2: Server Streaming
func testServerStreaming(ctx context.Context, client pb.ChatServiceClient) {
	log.Println("\n=== Pattern 2: Server Streaming ===")
	
	stream, err := client.ListUsers(ctx, &pb.Empty{})
	if err != nil {
		log.Fatalf("ListUsers failed: %v", err)
	}
	
	for {
		user, err := stream.Recv()
		if err == io.EOF {
			break // Stream ended
		}
		if err != nil {
			log.Fatalf("stream error: %v", err)
		}
		log.Printf("Received user: %+v", user)
	}
}

// Pattern 3: Client Streaming
func testClientStreaming(ctx context.Context, client pb.ChatServiceClient) {
	log.Println("\n=== Pattern 3: Client Streaming ===")
	
	stream, err := client.SendMessages(ctx)
	if err != nil {
		log.Fatalf("SendMessages failed: %v", err)
	}
	
	// Send multiple messages
	messages := []string{"Hello", "World", "From", "Client"}
	for _, msg := range messages {
		if err := stream.Send(&pb.ChatMessage{
			From:    "Client",
			Message: msg,
		}); err != nil {
			log.Fatalf("send error: %v", err)
		}
		time.Sleep(200 * time.Millisecond)
	}
	
	// Close and get response
	res, err := stream.CloseAndRecv()
	if err != nil {
		log.Fatalf("close error: %v", err)
	}
	log.Printf("Server received %d messages", res.Count)
}

// Pattern 4: Bidirectional
func testBidirectional(ctx context.Context, client pb.ChatServiceClient) {
	log.Println("\n=== Pattern 4: Bidirectional ===")
	
	stream, err := client.Chat(ctx)
	if err != nil {
		log.Fatalf("Chat failed: %v", err)
	}
	
	// Goroutine to receive messages
	done := make(chan bool)
	go func() {
		for {
			msg, err := stream.Recv()
			if err == io.EOF {
				done <- true
				return
			}
			if err != nil {
				log.Printf("recv error: %v", err)
				done <- true
				return
			}
			log.Printf("Server says: %s", msg.Message)
		}
	}()
	
	// Send messages
	messages := []string{"Hi!", "How are you?", "Bye!"}
	for _, msg := range messages {
		if err := stream.Send(&pb.ChatMessage{
			From:    "Keshav",
			Message: msg,
		}); err != nil {
			log.Fatalf("send error: %v", err)
		}
		time.Sleep(500 * time.Millisecond)
	}
	
	stream.CloseSend()
	<-done
}
```

---

### Task 5: Run and Test

**Terminal 1:**
```bash
cd /Users/keshavsharma/Grpc/phase-2-patterns/chat-service
go run server/main.go
```

**Terminal 2:**
```bash
cd /Users/keshavsharma/Grpc/phase-2-patterns/chat-service
go run client/main.go
```

**Expected Output:**
```
=== Pattern 1: Unary ===
Got user: id:"123" name:"User 123"

=== Pattern 2: Server Streaming ===
Received user: id:"1" name:"Keshav"
Received user: id:"2" name:"John"
Received user: id:"3" name:"Jane"

=== Pattern 3: Client Streaming ===
Server received 4 messages

=== Pattern 4: Bidirectional ===
Server says: Echo: Hi!
Server says: Echo: How are you?
Server says: Echo: Bye!
```

---

## ✅ Checklist

- [ ] Understand all 4 gRPC patterns
- [ ] Know when to use each pattern
- [ ] Implement server streaming (ListUsers)
- [ ] Implement client streaming (SendMessages)
- [ ] Implement bidirectional streaming (Chat)
- [ ] Successfully test all patterns

---

## 🔗 Next: [Phase 3 - REST to gRPC Migration](../phase-3-rest-migration/README.md)
