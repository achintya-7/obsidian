## Installation
To use gRPC with Go and make cool APIs, we have to use Protobuff and Protobuff Compillars. 
Follow the steps below to install and set it up

1. To install protobuff, use this command to install `protoc-gen-go` and `protoc-gen-go-grpc`
```sh
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

2. No to set path, use this command to get path of Go
```
go env GOPATH
```

Add this to your path variables
`%GOPATH%`  will look something like this `C:\Users\achin\go`
To set the PATH environment variable to include `%GOPATH%/bin` on Windows:
1.  Open the Start menu and search for "Environment Variables".
2.  Click on "Edit the system environment variables".
3.  Click on the "Environment Variables" button at the bottom of the "System Properties" window.
4.  Under "System Variables", scroll down and find the "Path" variable, then click on "Edit".
5.  Click on "New" and enter `%GOPATH%/bin` as the new path.
6.  Click "OK" to close all the windows.

3. Now check if they are successfully installed
```
protoc-gen-go --version && protoc-gen-go-grpc --version
```

## Protobuf
We will be using the `github.com/achintya-7/quiz` for this demo.
Install the first proto3 extension from VSCode to get syntax highlighting.

Make a dir called `proto` and add 3 files in it.
1. quiz.proto
2. rpc_create_quiz.proto
3. service_quiz.proto

### Quiz SQL migration
```sql
CREATE TABLE "quiz" (
  "id" bigserial PRIMARY KEY,
  "created_by" varchar NOT NULL,
  "name" varchar NOT NULL,
  "description" varchar NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT 'now()'
);
```

### quiz.proto
You will need to use protobuf timestamp module to use timestampz model
```proto3
syntax = "proto3";

package pb;

import "google/protobuf/timestamp.proto";

option go_package = "github.com/achintya-7/quiz/pb";

message Quiz {
    string id = 1;
    string created_by = 2;
    string name = 3;
    string description = 4;
    google.protobuf.Timestamp created_at = 5;
}
```

### rpc_create_quiz.proto
```
syntax = "proto3";

package pb;

import "quiz.proto";

option go_package = "github.com/achintya-7/quiz/pb";

message CreateQuizRequest {
    string name = 1;
    string description = 2;
    string created_by = 3;
}

message CreateQuizResponse {
    Quiz quiz = 1;
}
```

### service_quiz.proto
```
syntax = "proto3";

package pb;

import "rpc_create_quiz.proto";

option go_package = "github.com/achintya-7/quiz/pb";

service QuizService {
    rpc CreateQuiz (CreateQuizRequest) returns (CreateQuizResponse) {}
}
```

Now to run the protoc compilar

## Protoc compillar
We will get all the generated code in `pb` dir which is at the same scope of `proto`
```
protoc --proto_path=proto --go_out=pb --go_opt=paths=source_relative --go-grpc_out=pb --go-grpc_opt=paths=source_relative proto/*.proto
```

Let's understand this
1. `--proto_path=proto` : assign the path of proto dir
2. `--go_out=pb` : dumps the generated GO code in pb dir
3. `--go-grpc_out=pb` : dumps the generated gRPC code in pb dir
4. `proto/*.proto` : convert all the proto code here

## Creating gRPC server object
Create a `gapi` dir and create a `Server` struct
```go
package gapi

import (
	db "github.com/achintya-7/quiz/db/sqlc"
	"github.com/achintya-7/quiz/pb"
	"github.com/achintya-7/quiz/util"
)

type Server struct {
	pb.UnimplementedQuizServiceServer
	store  *db.Store
	config util.Config
}

func NewServer(store *db.Store, config util.Config) (*Server, error) {
	server := &Server{store: store, config: config}

	return server, nil
}
```

As the functions made with gRPC are made in a interface. If we don't define all the functions, it will throw an error. Thats why we add `pb.UnimplementedQuizServiceServer` in Server struct.
This will allows us to atleast start the server without all the functions implemented

Now in  `main.go` we can call and make a gRPC server 
```go
func main() {
	runGRPC(config, store)
}

func runGRPC(config util.Config, store *db.Store) {
	grpcServer := grpc.NewServer()
	server, err := gapi.NewServer(store, config)
	if err != nil {
		log.Fatal("Cannot create server", err)
	}

	pb.RegisterQuizServiceServer(grpcServer, server) // register function here

	reflection.Register(grpcServer)

	listener, err := net.Listen("tcp", config.GRPCServerAddress)
	if err != nil {
		log.Fatal("Cannot listen to port", err)
	}

	fmt.Println("GRPC server listening on port", config.GRPCServerAddress)

	err = grpcServer.Serve(listener)
	if err != nil {
		log.Fatal("Cannot start server", err)
	}

}
```

