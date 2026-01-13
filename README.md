# Proglog

A distributed, replicated log system built in Go. Proglog provides a high-performance, fault-tolerant log storage and retrieval system with strong consistency guarantees using the Raft consensus algorithm.

## Features

- **Distributed Log Storage**: Replicated log storage across multiple nodes for high availability
- **Raft Consensus**: Uses HashiCorp Raft for distributed consensus and leader election
- **Service Discovery**: Serf-based cluster membership and service discovery
- **gRPC API**: High-performance gRPC interface for produce/consume operations
- **Streaming Support**: Bidirectional streaming for real-time log consumption and production
- **TLS Security**: End-to-end encryption for both client-server and peer-to-peer communication
- **Access Control**: Casbin-based ACL (Access Control List) for fine-grained authorization
- **Memory-Mapped I/O**: Efficient log storage using memory-mapped files
- **Kubernetes Ready**: Helm charts for easy deployment on Kubernetes
- **Load Balancing**: Client-side load balancing with service discovery integration

## Architecture

Proglog is built with a modular architecture:

```
┌───────────────────────────────────────────────┐
│             Client Applications               │
└────────────────────┬──────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼────┐            ┌─────▼─────┐
    │  gRPC   │            │   HTTP    │
    │  API    │            │   Server  │
    └────┬────┘            └───────────┘
         │
    ┌────▼─────────────────────────────────┐
    │         Agent (Main Component)       │
    │  ┌──────────┐  ┌──────────────────┐  │
    │  │   Log    │  │   Membership     │  │
    │  │ (Raft)   │  │    (Serf)        │  │
    │  └──────────┘  └──────────────────┘  │
    └──────────────────────────────────────┘
```

### Core Components

- **Agent**: Main orchestrator that manages log, server, and membership
- **Log**: Distributed log implementation using Raft for consensus
- **Server**: gRPC and HTTP server implementations
- **Discovery**: Serf-based cluster membership management
- **Load Balance**: Client-side load balancing and service resolution
- **Auth**: Casbin-based authorization layer

## Prerequisites

- Go 1.25.5 or later
- Protocol Buffers compiler (`protoc`)
- Go plugins for protoc:
  - `google.golang.org/protobuf/cmd/protoc-gen-go`
  - `google.golang.org/grpc/cmd/protoc-gen-go-grpc`
- CFSSL (for certificate generation)
- Docker (optional, for containerized deployment)
- Kubernetes cluster (optional, for Kubernetes deployment)

## Installation

### From Source

1. Clone the repository:
```bash
git clone https://github.com/tabrizgulmammadov/proglog.git
cd proglog
```

2. Install dependencies:
```bash
go mod download
```

3. Generate protobuf code:
```bash
make compile
```

4. Build the binary:
```bash
go build -o bin/proglog ./cmd/proglog
```

### Docker

Build the Docker image:
```bash
make build-docker TAG=0.0.1
```

Or use Docker directly:
```bash
docker build -t github.com/tabrizgulmammadov/proglog:0.0.1 .
```

## Configuration

### Initial Setup

1. Initialize the configuration directory:
```bash
make init
```

2. Generate TLS certificates:
```bash
make gencert
```

This creates:
- CA certificate and key
- Server certificate and key
- Client certificates (root-client and nobody-client)

3. Copy ACL configuration files:
```bash
cp test/model.conf ${HOME}/.proglog/model.conf
cp test/policy.csv ${HOME}/.proglog/policy.csv
```

### Configuration Options

The `proglog` command supports the following flags:

| Flag | Description | Default |
|------|-------------|---------|
| `--config-file` | Path to config file | - |
| `--data-dir` | Directory to store log and Raft data | `/tmp/proglog` |
| `--node-name` | Unique server ID | Hostname |
| `--bind-addr` | Address to bind Serf on | `127.0.0.1:8401` |
| `--rpc-port` | Port for RPC clients (and Raft) | `8400` |
| `--start-join-addrs` | Serf addresses to join | - |
| `--bootstrap` | Bootstrap the cluster | `false` |
| `--acl-model-file` | Path to ACL model | - |
| `--acl-policy-file` | Path to ACL policy | - |
| `--server-tls-cert-file` | Path to server TLS cert | - |
| `--server-tls-key-file` | Path to server TLS key | - |
| `--server-tls-ca-file` | Path to server CA | - |
| `--peer-tls-cert-file` | Path to peer TLS cert | - |
| `--peer-tls-key-file` | Path to peer TLS key | - |
| `--peer-tls-ca-file` | Path to peer CA | - |

## Usage

### Starting a Single Node

```bash
./bin/proglog \
  --data-dir /tmp/proglog/node1 \
  --node-name node1 \
  --bind-addr 127.0.0.1:8401 \
  --rpc-port 8400 \
  --bootstrap
```

### Starting a Cluster

**Node 1 (Bootstrap):**
```bash
./bin/proglog \
  --data-dir /tmp/proglog/node1 \
  --node-name node1 \
  --bind-addr 127.0.0.1:8401 \
  --rpc-port 8400 \
  --bootstrap
```

**Node 2:**
```bash
./bin/proglog \
  --data-dir /tmp/proglog/node2 \
  --node-name node2 \
  --bind-addr 127.0.0.1:8402 \
  --rpc-port 8401 \
  --start-join-addrs 127.0.0.1:8401
```

**Node 3:**
```bash
./bin/proglog \
  --data-dir /tmp/proglog/node3 \
  --node-name node3 \
  --bind-addr 127.0.0.1:8403 \
  --rpc-port 8402 \
  --start-join-addrs 127.0.0.1:8401,127.0.0.1:8402
```

### With TLS

```bash
./bin/proglog \
  --data-dir /tmp/proglog/node1 \
  --node-name node1 \
  --bind-addr 127.0.0.1:8401 \
  --rpc-port 8400 \
  --server-tls-cert-file ${HOME}/.proglog/server.pem \
  --server-tls-key-file ${HOME}/.proglog/server-key.pem \
  --server-tls-ca-file ${HOME}/.proglog/ca.pem \
  --peer-tls-cert-file ${HOME}/.proglog/server.pem \
  --peer-tls-key-file ${HOME}/.proglog/server-key.pem \
  --peer-tls-ca-file ${HOME}/.proglog/ca.pem \
  --bootstrap
```

### Get Server List

Use the `getservers` command to query cluster membership:

```bash
go run ./cmd/getservers --addr localhost:8400
```

## API

Proglog exposes a gRPC API defined in `api/v1/log.proto`:

### Service Methods

- **Produce**: Append a record to the log
- **Consume**: Read a record at a specific offset
- **ConsumeStream**: Stream records starting from an offset
- **ProduceStream**: Bidirectional streaming for producing records
- **GetServers**: Get the list of servers in the cluster

### Example Client Usage

```go
import (
    "context"
    api "github.com/tabrizgulmammadov/proglog/api/v1"
    "google.golang.org/grpc"
)

conn, _ := grpc.Dial("localhost:8400", grpc.WithInsecure())
client := api.NewLogClient(conn)

// Produce a record
produceResp, _ := client.Produce(context.Background(), &api.ProduceRequest{
    Record: &api.Record{
        Value: []byte("hello world"),
    },
})

// Consume a record
consumeResp, _ := client.Consume(context.Background(), &api.ConsumeRequest{
    Offset: produceResp.Offset,
})
```

## Development

### Running Tests

```bash
make test
```

This will:
1. Copy ACL configuration files to `${HOME}/.proglog/`
2. Run all tests with race detection

### Project Structure

```
proglog/
├── api/v1/              # Protocol buffer definitions and generated code
├── cmd/
│   ├── proglog/         # Main server command
│   ├── server/          # HTTP server example
│   └── getservers/      # Server discovery client
├── internal/
│   ├── agent/           # Main agent orchestrator
│   ├── auth/            # Authorization layer (Casbin)
│   ├── config/          # Configuration management
│   ├── discovery/       # Serf-based service discovery
│   ├── loadbalance/     # Client-side load balancing
│   ├── log/             # Distributed log implementation
│   └── server/          # gRPC and HTTP servers
├── deploy/              # Kubernetes deployment manifests
│   ├── proglog/         # Proglog Helm chart
│   └── metacontroller/  # Metacontroller Helm chart
└── test/                # Test certificates and ACL configs
```

### Code Generation

Generate protobuf code:
```bash
make compile
```

## Deployment

### Kubernetes

Deploy using Helm:

```bash
# Install the chart
helm install proglog ./deploy/proglog

# Or with custom values
helm install proglog ./deploy/proglog -f custom-values.yaml
```

The Helm chart includes:
- StatefulSet for ordered, stable deployment
- Service per pod for service discovery
- ConfigMaps and Secrets support
- Health checks and readiness probes

### Docker Compose

Example `docker-compose.yml`:

```yaml
version: '3.8'
services:
  node1:
    image: github.com/tabrizgulmammadov/proglog:0.0.1
    ports:
      - "8401:8401"
      - "8400:8400"
    command: ["--bootstrap", "--data-dir", "/data"]
    volumes:
      - node1-data:/data

  node2:
    image: github.com/tabrizgulmammadov/proglog:0.0.1
    ports:
      - "8402:8401"
      - "8401:8400"
    command: ["--start-join-addrs", "node1:8401", "--data-dir", "/data"]
    volumes:
      - node2-data:/data
    depends_on:
      - node1

volumes:
  node1-data:
  node2-data:
```

## Security

- **TLS Encryption**: All client-server and peer-to-peer communication can be encrypted with TLS
- **ACL Authorization**: Fine-grained access control using Casbin policies
- **Certificate Management**: CFSSL-based certificate generation and management

## Performance Considerations

- **Memory-Mapped I/O**: Uses memory-mapped files for efficient log storage
- **Segment-Based Storage**: Logs are stored in segments for efficient rotation and compaction
- **Raft Batching**: Raft operations are batched for better throughput
- **Connection Multiplexing**: Uses cmux for multiplexing gRPC and Raft on the same port

## Troubleshooting

### Common Issues

1. **Port conflicts**: Ensure ports 8400 (RPC) and 8401 (Serf) are available
2. **Certificate errors**: Verify TLS certificates are correctly generated and paths are correct
3. **Cluster formation**: Ensure at least one bootstrap node is started before other nodes join
4. **Data directory permissions**: Ensure the data directory is writable
5. **Config file format errors**: In Kubernetes, ensure `start-join-addrs` is formatted as a YAML array: `["addr1:port", "addr2:port"]` not as a string

### Kubernetes Debugging

If pods are failing in Kubernetes:

1. **Check pod logs**:
```bash
kubectl logs proglog-0
```

2. **Check the generated config file**:
```bash
kubectl exec proglog-0 -- cat /var/run/proglog/config.yaml
```

3. **Verify the config file format** - The `start-join-addrs` should be a YAML array:
```yaml
start-join-addrs: ["proglog-0.proglog.default.svc.cluster.local:8401"]
```
Not a string:
```yaml
start-join-addrs: "proglog-0.proglog.default.svc.cluster.local:8401"  # WRONG
```

4. **Check if the data directory exists and is writable**:
```bash
kubectl exec proglog-0 -- ls -la /var/run/proglog/
```

5. **Describe the pod for events**:
```bash
kubectl describe pod proglog-0
```

6. **Check if the service is accessible**:
```bash
kubectl get svc
kubectl get endpoints
```

### Debugging

Enable debug logging by setting the log level in the agent configuration or using environment variables.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

See [LICENSE](LICENSE) file for details.
