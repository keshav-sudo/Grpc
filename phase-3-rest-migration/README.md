# Phase 3: REST to gRPC Migration 🔄

## What You'll Learn
- Map REST endpoints to gRPC methods
- Convert JSON request/response to Protobuf
- Handle errors (HTTP codes → gRPC status)
- When to migrate vs when to keep REST

---

## 📖 Theory

### REST vs gRPC Mapping

| REST | gRPC |
|------|------|
| `GET /users/:id` | `rpc GetUser(GetUserRequest) returns (User)` |
| `POST /users` | `rpc CreateUser(CreateUserRequest) returns (User)` |
| `PUT /users/:id` | `rpc UpdateUser(UpdateUserRequest) returns (User)` |
| `DELETE /users/:id` | `rpc DeleteUser(DeleteUserRequest) returns (Empty)` |
| `GET /users` | `rpc ListUsers(ListUsersRequest) returns (ListUsersResponse)` |

### REST JSON → Protobuf Message

**REST Request:**
```json
POST /users
{
  "name": "Keshav",
  "email": "keshav@example.com",
  "age": 25,
  "tags": ["developer", "golang"]
}
```

**Protobuf Message:**
```protobuf
message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
  repeated string tags = 4;   // array
}
```

### Error Handling: HTTP → gRPC

| HTTP Status | gRPC Status Code | When to Use |
|-------------|------------------|-------------|
| 200 OK | `codes.OK` | Success |
| 400 Bad Request | `codes.InvalidArgument` | Bad input |
| 401 Unauthorized | `codes.Unauthenticated` | No/bad token |
| 403 Forbidden | `codes.PermissionDenied` | Not allowed |
| 404 Not Found | `codes.NotFound` | Resource missing |
| 409 Conflict | `codes.AlreadyExists` | Duplicate |
| 500 Server Error | `codes.Internal` | Server bug |
| 503 Unavailable | `codes.Unavailable` | Service down |

**In Go code:**
```go
import "google.golang.org/grpc/codes"
import "google.golang.org/grpc/status"

// Return error
return nil, status.Errorf(codes.NotFound, "user %d not found", id)

// Check error in client
if status.Code(err) == codes.NotFound {
    // Handle not found
}
```

---

## 💻 Tasks

### The Scenario

You have a REST API (like in Node.js). We'll convert it to gRPC.

**Original REST API (Node.js style):**
```
GET    /api/products           → List all products
GET    /api/products/:id       → Get one product
POST   /api/products           → Create product
PUT    /api/products/:id       → Update product
DELETE /api/products/:id       → Delete product
GET    /api/products/search?q= → Search products (returns stream)
```

---

### Task 1: Create Proto File

Create `/Users/keshavsharma/Grpc/phase-3-rest-migration/api-migration/proto/product.proto`:

```protobuf
syntax = "proto3";

package product;

option go_package = "./pb";

// ========== Messages ==========

message Product {
  int32 id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  int32 stock = 5;
  repeated string categories = 6;
  int64 created_at = 7;
  int64 updated_at = 8;
}

// REST: GET /products/:id
message GetProductRequest {
  int32 id = 1;
}

// REST: POST /products
message CreateProductRequest {
  string name = 1;
  string description = 2;
  double price = 3;
  int32 stock = 4;
  repeated string categories = 5;
}

// REST: PUT /products/:id
message UpdateProductRequest {
  int32 id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  int32 stock = 5;
  repeated string categories = 6;
}

// REST: DELETE /products/:id
message DeleteProductRequest {
  int32 id = 1;
}

// REST: GET /products
message ListProductsRequest {
  int32 page = 1;
  int32 limit = 2;
}

message ListProductsResponse {
  repeated Product products = 1;
  int32 total = 2;
}

// REST: GET /products/search?q=
message SearchProductsRequest {
  string query = 1;
}

message Empty {}

// ========== Service ==========

service ProductService {
  // CRUD Operations (Unary - like REST)
  rpc GetProduct(GetProductRequest) returns (Product);
  rpc CreateProduct(CreateProductRequest) returns (Product);
  rpc UpdateProduct(UpdateProductRequest) returns (Product);
  rpc DeleteProduct(DeleteProductRequest) returns (Empty);
  
  // List with pagination
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
  
  // Search with streaming (better than REST for large results)
  rpc SearchProducts(SearchProductsRequest) returns (stream Product);
}
```

---

### Task 2: Setup Project

```bash
cd /Users/keshavsharma/Grpc/phase-3-rest-migration
mkdir -p api-migration/{proto,pb,server,client}
cd api-migration

go mod init api-migration
go get google.golang.org/grpc
go get google.golang.org/protobuf

protoc --go_out=. --go-grpc_out=. proto/product.proto
```

---

### Task 3: Implement Server with Proper Errors

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
	"google.golang.org/grpc/status"
	pb "api-migration/pb"
)

type server struct {
	pb.UnimplementedProductServiceServer
	products map[int32]*pb.Product
	nextID   int32
	mu       sync.RWMutex
}

func newServer() *server {
	s := &server{
		products: make(map[int32]*pb.Product),
		nextID:   1,
	}
	// Seed data
	s.products[1] = &pb.Product{Id: 1, Name: "Laptop", Description: "Gaming laptop", Price: 999.99, Stock: 10, Categories: []string{"electronics"}}
	s.products[2] = &pb.Product{Id: 2, Name: "Phone", Description: "Smartphone", Price: 599.99, Stock: 25, Categories: []string{"electronics", "mobile"}}
	s.products[3] = &pb.Product{Id: 3, Name: "Headphones", Description: "Wireless", Price: 149.99, Stock: 50, Categories: []string{"electronics", "audio"}}
	s.nextID = 4
	return s
}

// GetProduct - like GET /products/:id
func (s *server) GetProduct(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	// Validation - like Express validation
	if req.Id <= 0 {
		return nil, status.Errorf(codes.InvalidArgument, "invalid product id: %d", req.Id)
	}
	
	product, exists := s.products[req.Id]
	if !exists {
		// 404 → NotFound
		return nil, status.Errorf(codes.NotFound, "product %d not found", req.Id)
	}
	
	return product, nil
}

// CreateProduct - like POST /products
func (s *server) CreateProduct(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	// Validation
	if req.Name == "" {
		return nil, status.Errorf(codes.InvalidArgument, "name is required")
	}
	if req.Price <= 0 {
		return nil, status.Errorf(codes.InvalidArgument, "price must be positive")
	}
	
	product := &pb.Product{
		Id:          s.nextID,
		Name:        req.Name,
		Description: req.Description,
		Price:       req.Price,
		Stock:       req.Stock,
		Categories:  req.Categories,
		CreatedAt:   time.Now().Unix(),
		UpdatedAt:   time.Now().Unix(),
	}
	
	s.products[s.nextID] = product
	s.nextID++
	
	log.Printf("Created product: %+v", product)
	return product, nil
}

// UpdateProduct - like PUT /products/:id
func (s *server) UpdateProduct(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	product, exists := s.products[req.Id]
	if !exists {
		return nil, status.Errorf(codes.NotFound, "product %d not found", req.Id)
	}
	
	// Update fields
	if req.Name != "" {
		product.Name = req.Name
	}
	if req.Description != "" {
		product.Description = req.Description
	}
	if req.Price > 0 {
		product.Price = req.Price
	}
	if req.Stock >= 0 {
		product.Stock = req.Stock
	}
	if len(req.Categories) > 0 {
		product.Categories = req.Categories
	}
	product.UpdatedAt = time.Now().Unix()
	
	log.Printf("Updated product: %+v", product)
	return product, nil
}

// DeleteProduct - like DELETE /products/:id
func (s *server) DeleteProduct(ctx context.Context, req *pb.DeleteProductRequest) (*pb.Empty, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	
	if _, exists := s.products[req.Id]; !exists {
		return nil, status.Errorf(codes.NotFound, "product %d not found", req.Id)
	}
	
	delete(s.products, req.Id)
	log.Printf("Deleted product: %d", req.Id)
	return &pb.Empty{}, nil
}

// ListProducts - like GET /products?page=1&limit=10
func (s *server) ListProducts(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	// Default pagination
	page := req.Page
	if page <= 0 {
		page = 1
	}
	limit := req.Limit
	if limit <= 0 {
		limit = 10
	}
	
	// Convert map to slice
	var allProducts []*pb.Product
	for _, p := range s.products {
		allProducts = append(allProducts, p)
	}
	
	// Paginate
	start := (page - 1) * limit
	end := start + limit
	if start >= int32(len(allProducts)) {
		return &pb.ListProductsResponse{Products: []*pb.Product{}, Total: int32(len(allProducts))}, nil
	}
	if end > int32(len(allProducts)) {
		end = int32(len(allProducts))
	}
	
	return &pb.ListProductsResponse{
		Products: allProducts[start:end],
		Total:    int32(len(allProducts)),
	}, nil
}

// SearchProducts - like GET /products/search?q= but STREAMING
func (s *server) SearchProducts(req *pb.SearchProductsRequest, stream pb.ProductService_SearchProductsServer) error {
	s.mu.RLock()
	defer s.mu.RUnlock()
	
	query := strings.ToLower(req.Query)
	
	for _, product := range s.products {
		// Check if matches
		if strings.Contains(strings.ToLower(product.Name), query) ||
			strings.Contains(strings.ToLower(product.Description), query) {
			
			if err := stream.Send(product); err != nil {
				return err
			}
			time.Sleep(100 * time.Millisecond) // Simulate search delay
		}
	}
	
	return nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	
	s := grpc.NewServer()
	pb.RegisterProductServiceServer(s, newServer())
	
	log.Println("Product gRPC server running on :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

---

### Task 4: Implement Client with Error Handling

Create `client/main.go`:

```go
package main

import (
	"context"
	"io"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
	pb "api-migration/pb"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewProductServiceClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// Test CRUD operations
	testGetProduct(ctx, client)
	testCreateProduct(ctx, client)
	testListProducts(ctx, client)
	testSearchProducts(ctx, client)
	testUpdateProduct(ctx, client)
	testDeleteProduct(ctx, client)
	testErrorHandling(ctx, client)
}

func testGetProduct(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== GET /products/1 ===")
	
	product, err := client.GetProduct(ctx, &pb.GetProductRequest{Id: 1})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Product: %+v", product)
}

func testCreateProduct(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== POST /products ===")
	
	product, err := client.CreateProduct(ctx, &pb.CreateProductRequest{
		Name:        "Keyboard",
		Description: "Mechanical keyboard",
		Price:       79.99,
		Stock:       100,
		Categories:  []string{"electronics", "computer"},
	})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Created: %+v", product)
}

func testListProducts(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== GET /products?page=1&limit=10 ===")
	
	res, err := client.ListProducts(ctx, &pb.ListProductsRequest{Page: 1, Limit: 10})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Total: %d products", res.Total)
	for _, p := range res.Products {
		log.Printf("  - %s ($%.2f)", p.Name, p.Price)
	}
}

func testSearchProducts(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== GET /products/search?q=phone (STREAMING) ===")
	
	stream, err := client.SearchProducts(ctx, &pb.SearchProductsRequest{Query: "phone"})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	
	for {
		product, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Printf("Stream error: %v", err)
			return
		}
		log.Printf("Found: %s - %s", product.Name, product.Description)
	}
}

func testUpdateProduct(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== PUT /products/1 ===")
	
	product, err := client.UpdateProduct(ctx, &pb.UpdateProductRequest{
		Id:    1,
		Price: 899.99, // Discount!
	})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Printf("Updated: %s now $%.2f", product.Name, product.Price)
}

func testDeleteProduct(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== DELETE /products/3 ===")
	
	_, err := client.DeleteProduct(ctx, &pb.DeleteProductRequest{Id: 3})
	if err != nil {
		log.Printf("Error: %v", err)
		return
	}
	log.Println("Deleted product 3")
}

func testErrorHandling(ctx context.Context, client pb.ProductServiceClient) {
	log.Println("\n=== Error Handling Demo ===")
	
	// Try to get non-existent product
	_, err := client.GetProduct(ctx, &pb.GetProductRequest{Id: 999})
	if err != nil {
		st := status.Convert(err)
		log.Printf("Error Code: %s", st.Code())
		log.Printf("Error Message: %s", st.Message())
		
		// Handle specific errors (like in Express error middleware)
		switch st.Code() {
		case codes.NotFound:
			log.Println("→ Product not found (like 404)")
		case codes.InvalidArgument:
			log.Println("→ Bad request (like 400)")
		case codes.Internal:
			log.Println("→ Server error (like 500)")
		}
	}
}
```

---

### Task 5: Run and Test

**Terminal 1:**
```bash
cd /Users/keshavsharma/Grpc/phase-3-rest-migration/api-migration
go run server/main.go
```

**Terminal 2:**
```bash
cd /Users/keshavsharma/Grpc/phase-3-rest-migration/api-migration
go run client/main.go
```

---

## 📊 REST vs gRPC Comparison

```
REST (Node.js)                    gRPC (Go)
--------------                    ---------
app.get('/products/:id')    →     rpc GetProduct(req) returns (Product)
res.json(product)           →     return product, nil
res.status(404).json(...)   →     status.Errorf(codes.NotFound, ...)
JSON parsing overhead       →     Binary protobuf (7-10x faster)
Manual API docs             →     Auto-generated from .proto
```

---

## ✅ Checklist

- [ ] Map REST endpoints to gRPC methods
- [ ] Create Protobuf messages for request/response
- [ ] Implement CRUD with proper error codes
- [ ] Handle errors using gRPC status codes
- [ ] Use streaming for search results
- [ ] Test all endpoints work correctly

---

## 🔗 Next: [Phase 4 - Production Ready](../phase-4-production/README.md)
