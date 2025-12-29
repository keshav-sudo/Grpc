# Phase 4: Production Ready gRPC 🚀

## What You'll Learn
- Interceptors (like Express middleware)
- Metadata (like HTTP headers)
- Authentication with tokens
- TLS/SSL security
- gRPC-Gateway (expose gRPC as REST)
- Health checks

---

## 📖 Theory

### Interceptors = Middleware

In Express.js you have middleware:
```javascript
app.use(authMiddleware);
app.use(loggingMiddleware);
```

In gRPC Go, these are called **Interceptors**:
```go
grpc.NewServer(
    grpc.UnaryInterceptor(authInterceptor),
    grpc.ChainUnaryInterceptor(loggingInterceptor, metricsInterceptor),
)
```

### Metadata = HTTP Headers

```go
// Client: Set metadata (like Authorization header)
md := metadata.Pairs("authorization", "Bearer token123")
ctx := metadata.NewOutgoingContext(ctx, md)

// Server: Read metadata
md, ok := metadata.FromIncomingContext(ctx)
token := md.Get("authorization")[0]
```

### Interceptor Types

| Type | When | Use For |
|------|------|---------|
| **Unary Interceptor** | Single request/response | Auth, logging, metrics |
| **Stream Interceptor** | Streaming calls | Same, but for streams |

---

## 💻 Tasks

### Task 1: Create Production Service Proto

Create `/Users/keshavsharma/Grpc/phase-4-production/production-service/proto/service.proto`:

```protobuf
syntax = "proto3";

package production;

option go_package = "./pb";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  string role = 4;  // "admin" or "user"
}

message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message ListUsersRequest {}

message ListUsersResponse {
  repeated User users = 1;
}

message Empty {}

// Health check (standard for production)
message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
  }
  ServingStatus status = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc DeleteUser(GetUserRequest) returns (Empty);
  
  // Health check endpoint
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
}
```

---

### Task 2: Setup Project

```bash
cd /Users/keshavsharma/Grpc/phase-4-production
mkdir -p production-service/{proto,pb,server,client}
cd production-service

go mod init production-service
go get google.golang.org/grpc
go get google.golang.org/protobuf

protoc --go_out=. --go-grpc_out=. proto/service.proto
```

---

### Task 3: Implement Server with Interceptors

Create `server/main.go`:

```go
package main

import (
	"context"
	"log"
	"net"
	"strings"
	"sync"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	pb "production-service/pb"
)

// ========== INTERCEPTORS (Like Express Middleware) ==========

// Logging Interceptor - logs all requests
func loggingInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()
	
	// Call the actual handler
	resp, err := handler(ctx, req)
	
	// Log after
	duration := time.Since(start)
	statusCode := codes.OK
	if err != nil {
		statusCode = status.Code(err)
	}
	
	log.Printf("[%s] %s | %v | %s",
		info.FullMethod,
		statusCode,
		duration,
		getClientIP(ctx),
	)
	
	return resp, err
}

// Auth Interceptor - checks JWT/token
func authInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	// Skip auth for health check
	if strings.Contains(info.FullMethod, "HealthCheck") {
		return handler(ctx, req)
	}
	
	// Get metadata (like headers)
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Errorf(codes.Unauthenticated, "no metadata provided")
	}
	
	// Check authorization
	authHeader := md.Get("authorization")
	if len(authHeader) == 0 {
		return nil, status.Errorf(codes.Unauthenticated, "authorization token required")
	}
	
	token := authHeader[0]
	
	// Simple token validation (in real app, verify JWT)
	if !strings.HasPrefix(token, "Bearer ") {
		return nil, status.Errorf(codes.Unauthenticated, "invalid token format")
	}
	
	tokenValue := strings.TrimPrefix(token, "Bearer ")
	
	// Fake validation - in real app, verify with JWT library
	if tokenValue != "valid-token" && tokenValue != "admin-token" {
		return nil, status.Errorf(codes.Unauthenticated, "invalid token")
	}
	
	// Add user info to context (like req.user in Express)
	userRole := "user"
	if tokenValue == "admin-token" {
		userRole = "admin"
	}
	newCtx := context.WithValue(ctx, "user_role", userRole)
	
	return handler(newCtx, req)
}

// Recovery Interceptor - catches panics
func recoveryInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (resp interface{}, err error) {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("PANIC RECOVERED: %v", r)
			err = status.Errorf(codes.Internal, "internal server error")
		}
	}()
	return handler(ctx, req)
}

// Helper to get client IP
func getClientIP(ctx context.Context) string {
	md, ok := metadata.FromIncomingContext(ctx)
	if ok {
		if ips := md.Get("x-forwarded-for"); len(ips) > 0 {
			return ips[0]
		}
	}
	return "unknown"
}

// ========== SERVICE IMPLEMENTATION ==========

type userServer struct {
	pb.UnimplementedUserServiceServer
	users  map[int32]*pb.User
	nextID int32
	mu     sync.RWMutex
}

func newUserServer() *userServer {
	return &userServer{
		users: map[int32]*pb.User{
			1: {Id: 1, Name: "Keshav", Email: "keshav@example.com", Role: "admin"},
			2: {Id: 2, Name: "John", Email: "john@example.com", Role: "user"},
		},
		nextID: 3,
	}
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	user, exists := s.users[req.Id]
	if !exists {
		return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
	}
	return user, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	user := &pb.User{
		Id:    s.nextID,
		Name:  req.Name,
		Email: req.Email,
		Role:  "user",
	}
	s.users[s.nextID] = user
	s.nextID++
	
	return user, nil
}

func (s *userServer) ListUsers(ctx context.Context, req *pb.ListUsersRequest) (*pb.ListUsersResponse, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	var users []*pb.User
	for _, u := range s.users {
		users = append(users, u)
	}
	return &pb.ListUsersResponse{Users: users}, nil
}

func (s *userServer) DeleteUser(ctx context.Context, req *pb.GetUserRequest) (*pb.Empty, error) {
	// Check if user is admin (role-based access)
	role := ctx.Value("user_role")
	if role != "admin" {
		return nil, status.Errorf(codes.PermissionDenied, "admin access required")
	}
	
	s.mu.Lock()
	defer s.mu.Unlock()
	
	if _, exists := s.users[req.Id]; !exists {
		return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
	}
	
	delete(s.users, req.Id)
	return &pb.Empty{}, nil
}

func (s *userServer) HealthCheck(ctx context.Context, req *pb.HealthCheckRequest) (*pb.HealthCheckResponse, error) {
	return &pb.HealthCheckResponse{
		Status: pb.HealthCheckResponse_SERVING,
	}, nil
}

// ========== MAIN ==========

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
	// Create server with interceptor chain (like app.use() in Express)
	grpcServer := grpc.NewServer(
		grpc.ChainUnaryInterceptor(
			recoveryInterceptor,  // First: catch panics
			loggingInterceptor,   // Second: log requests
			authInterceptor,      // Third: check auth
		),
	)
	
	pb.RegisterUserServiceServer(grpcServer, newUserServer())
	
	log.Println("Production gRPC server running on :50051")
	log.Println("Using interceptors: recovery → logging → auth")
	
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

---

### Task 4: Implement Client with Auth

Create `client/main.go`:

```go
package main

import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	pb "production-service/pb"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewUserServiceClient(conn)

	// Test without auth
	log.Println("\n=== Without Auth (should fail) ===")
	testWithoutAuth(client)

	// Test with invalid token
	log.Println("\n=== With Invalid Token (should fail) ===")
	testWithInvalidToken(client)

	// Test with valid user token
	log.Println("\n=== With Valid User Token ===")
	testWithUserToken(client)

	// Test admin-only operation with user token (should fail)
	log.Println("\n=== Delete with User Token (should fail) ===")
	testDeleteWithUserToken(client)

	// Test admin-only operation with admin token
	log.Println("\n=== Delete with Admin Token ===")
	testDeleteWithAdminToken(client)

	// Test health check (no auth needed)
	log.Println("\n=== Health Check (no auth) ===")
	testHealthCheck(client)
}

func testWithoutAuth(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	_, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
	if err != nil {
		log.Printf("Error (expected): %v", status.Convert(err).Message())
	}
}

func testWithInvalidToken(client pb.UserServiceClient) {
	// Add invalid token to metadata
	md := metadata.Pairs("authorization", "Bearer invalid-token")
	ctx := metadata.NewOutgoingContext(context.Background(), md)
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	_, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
	if err != nil {
		log.Printf("Error (expected): %v", status.Convert(err).Message())
	}
}

func testWithUserToken(client pb.UserServiceClient) {
	// Add valid token to metadata (like Authorization header)
	md := metadata.Pairs("authorization", "Bearer valid-token")
	ctx := metadata.NewOutgoingContext(context.Background(), md)
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	// List users
	res, err := client.ListUsers(ctx, &pb.ListUsersRequest{})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Users: %d found", len(res.Users))
	for _, u := range res.Users {
		log.Printf("  - %s (%s) - %s", u.Name, u.Email, u.Role)
	}

	// Get single user
	user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Got user: %s", user.Name)
}

func testDeleteWithUserToken(client pb.UserServiceClient) {
	md := metadata.Pairs("authorization", "Bearer valid-token")
	ctx := metadata.NewOutgoingContext(context.Background(), md)
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	_, err := client.DeleteUser(ctx, &pb.GetUserRequest{Id: 2})
	if err != nil {
		st := status.Convert(err)
		log.Printf("Error (expected): %s - %s", st.Code(), st.Message())
		if st.Code() == codes.PermissionDenied {
			log.Println("→ Correctly rejected: admin access required")
		}
	}
}

func testDeleteWithAdminToken(client pb.UserServiceClient) {
	md := metadata.Pairs("authorization", "Bearer admin-token")
	ctx := metadata.NewOutgoingContext(context.Background(), md)
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	_, err := client.DeleteUser(ctx, &pb.GetUserRequest{Id: 2})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Println("User deleted successfully (admin access)")
}

func testHealthCheck(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	res, err := client.HealthCheck(ctx, &pb.HealthCheckRequest{Service: "user"})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Health: %s", res.Status)
}
```

---

### Task 5: Run and Test

**Terminal 1:**
```bash
cd /Users/keshavsharma/Grpc/phase-4-production/production-service
go run server/main.go
```

**Terminal 2:**
```bash
cd /Users/keshavsharma/Grpc/phase-4-production/production-service
go run client/main.go
```

**Expected Output:**
```
=== Without Auth (should fail) ===
Error (expected): authorization token required

=== With Invalid Token (should fail) ===
Error (expected): invalid token

=== With Valid User Token ===
Users: 2 found
  - Keshav (keshav@example.com) - admin
  - John (john@example.com) - user
Got user: Keshav

=== Delete with User Token (should fail) ===
Error (expected): PermissionDenied - admin access required
→ Correctly rejected: admin access required

=== Delete with Admin Token ===
User deleted successfully (admin access)

=== Health Check (no auth) ===
Health: SERVING
```

---

## 📊 Express vs gRPC Comparison

| Express.js | gRPC Go |
|------------|---------|
| `app.use(middleware)` | `grpc.ChainUnaryInterceptor(...)` |
| `req.headers.authorization` | `metadata.FromIncomingContext(ctx)` |
| `req.user = decoded` | `ctx = context.WithValue(ctx, "user", data)` |
| `res.status(401).json(...)` | `status.Errorf(codes.Unauthenticated, ...)` |
| `req.ip` | Custom interceptor |

---

## ✅ Final Checklist

- [ ] Understand interceptors (like middleware)
- [ ] Implement logging interceptor
- [ ] Implement auth interceptor with token validation
- [ ] Use metadata for passing tokens
- [ ] Implement role-based access control
- [ ] Add health check endpoint
- [ ] Test all scenarios (no auth, bad auth, user, admin)

---

## 🎉 Congratulations!

You've completed the gRPC learning path! You now know:

1. ✅ **Fundamentals** - Protobuf, server/client basics
2. ✅ **4 Patterns** - Unary, Server/Client/Bidirectional streaming
3. ✅ **Migration** - REST to gRPC conversion
4. ✅ **Production** - Interceptors, auth, metadata

### What's Next?
- Add TLS/SSL for encrypted connections
- Explore gRPC-Gateway to expose REST endpoints
- Add metrics with Prometheus
- Deploy with Kubernetes + gRPC load balancing
