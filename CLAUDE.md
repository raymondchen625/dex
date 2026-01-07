# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dex is a federated OpenID Connect (OIDC) provider that acts as an identity service portal. It uses "connectors" to defer authentication to upstream identity providers (LDAP, SAML, GitHub, Google, etc.) while exposing a standard OIDC interface to client applications. This allows clients to write authentication logic once against Dex rather than integrating with multiple identity providers.

## Development Commands

### Build

```bash
# Build the main dex binary
make build  # Output: bin/dex

# Build example applications
make examples  # Output: bin/grpc-client, bin/example-app

# Build release binaries (with static linking)
make release-binary
```

### Testing

```bash
# Run all tests
make test

# Run tests with race detection
make testrace

# Run all tests (same as testrace)
make testall

# Run tests on a kind cluster (requires cluster setup first)
make kind-up     # Create kind cluster
make kind-tests  # Run tests on cluster
make kind-down   # Cleanup cluster
```

### Code Generation

```bash
# Generate all code (protobuf, ent ORM, go mod tidy)
make generate

# Generate only protobuf client API code
make generate-proto

# Generate internal protobuf code (token encoding)
make generate-proto-internal

# Generate database ORM code (ent)
make generate-ent

# Tidy all go modules (root, examples, api/v2)
make go-mod-tidy
```

### Linting and Formatting

```bash
# Run linter
make lint

# Fix lint violations automatically
make fix

# Verify all generated code is committed
make verify
```

### Development Environment

```bash
# Install all development dependencies
make deps

# Launch development environment with docker-compose
make up

# Destroy development environment
make down
```

### Running Dex

```bash
# Run dex server with config file
bin/dex serve config.yaml

# Or directly with go
go run cmd/dex/main.go serve config.yaml
```

## Architecture Overview

### Core Architecture Pattern

Dex implements a **federated identity bridge** with three key layers:

1. **HTTP Server Layer** (`/server/`): Handles OAuth2/OIDC protocol flows (authorization, token exchange, callbacks)
2. **Connector Layer** (`/connector/`): Pluggable authentication strategies for upstream identity providers
3. **Storage Layer** (`/storage/`): Abstracted persistence with multiple backend implementations

### Two-Server Design

Dex runs two separate servers:
- **HTTP Server** (default `:5556`): Public OAuth2/OIDC endpoints for client applications
- **gRPC Server** (default `:5557`): Management API for administrative operations (CRUD for clients, connectors, passwords)

This separation ensures user-facing authentication flows are isolated from administrative operations.

### Key Directories

- **`cmd/dex/`**: Main application entry point, configuration loading, server initialization
- **`server/`**: Core HTTP handlers for OAuth2/OIDC flows, gRPC API implementation, template management
- **`connector/`**: Identity provider integrations (each subdirectory is a different connector type)
  - Connectors implement different interfaces based on authentication method:
    - `CallbackConnector`: OAuth2-style redirects (GitHub, Google, OIDC)
    - `PasswordConnector`: Username/password (LDAP, local passwords)
    - `SAMLConnector`: SAML POST binding
    - `RefreshConnector`: Optional refresh token support
- **`storage/`**: Storage abstraction layer with multiple backends
  - `storage.go`: Core interface definition
  - `sql/`, `memory/`, `kubernetes/`, `etcd/`: Backend implementations
  - `ent/`: Alternative ORM-based SQL backend
  - `conformance/`: Test suite for storage backends
- **`api/v2/`**: gRPC API definitions (protobuf) and generated code
- **`pkg/`**: Shared utilities (feature flags, HTTP client, group handling)

### Connector Architecture

Connectors use **opaque ConnectorData** (`[]byte`) to cache upstream provider state:
- Stored in: `AuthRequest` → `AuthCode` → `RefreshToken`
- Allows connectors to store access tokens, user IDs without Dex knowing internals
- Enables efficient refresh operations without re-prompting user

Connectors are **hot-reloadable**: Server checks storage ResourceVersion and reloads if changed.

### Storage Interface

The storage interface is simple CRUD (no complex queries) to support diverse backends:
- Update operations use `updater func(old T) (T, error)` pattern for optimistic concurrency
- Backends handle their own indexing, transactions, and consistency
- This simplicity enables SQL, Kubernetes CRDs, etcd, and in-memory backends

All state is in shared storage, making Dex **horizontally scalable** with no sticky sessions.

### Authorization Flow

1. Client requests `/authorize` → Dex creates `AuthRequest`, redirects to connector
2. User authenticates with upstream provider (GitHub, LDAP, etc.)
3. Upstream redirects to `/callback` → Connector extracts identity (email, groups, etc.)
4. User approves scopes → Dex creates `AuthCode` with embedded claims
5. Client exchanges code at `/token` → Dex issues signed JWT ID token
6. Refresh tokens (if supported) allow updating claims without re-authentication

### Token Management

- **Signing**: RS256 (RSA-SHA256) only
- **Key Rotation**: Automatic rotation (default 6 hours), old keys kept for verification
- **Refresh Tokens**: Rotated on each use (old token invalidated)
- **PKCE Support**: Proof Key for Code Exchange (RFC 7636) for public clients

## Configuration

Dex uses YAML configuration with these key sections:

```yaml
# Core identity
issuer: https://dex.example.com

# Storage backend (postgres, sqlite3, kubernetes, etcd, memory)
storage:
  type: postgres
  config:
    # Backend-specific configuration

# HTTP endpoints
web:
  http: 0.0.0.0:5556
  https: 0.0.0.0:5557
  tlsCert: /path/to/cert.pem
  tlsKey: /path/to/key.pem

# gRPC management API
grpc:
  addr: 0.0.0.0:5558
  tlsCert: /path/to/cert.pem
  tlsKey: /path/to/key.pem

# Token expiry settings
expiry:
  idTokens: "24h"
  authRequests: "24h"
  refreshTokens: "24h"

# Upstream identity providers
connectors:
  - type: github
    id: github
    name: "GitHub"
    config:
      clientID: <id>
      clientSecret: <secret>
      redirectURI: https://dex.example.com/callback

# OAuth2 clients
staticClients:
  - id: app1
    secret: <secret>
    redirectURIs:
      - http://localhost:3000/callback

# Local password database (optional)
enablePasswordDB: true
```

## Module Structure

The repository has three Go modules:
- **Root module** (`go.mod`): Main Dex application
- **`api/v2/go.mod`**: gRPC API client library (independent versioning)
- **`examples/go.mod`**: Example applications

When updating dependencies, run `make go-mod-tidy` to update all three modules.

## Extending Dex

### Adding a New Connector

1. Create package: `connector/myidp/`
2. Implement `Config` struct with `Open(id string, logger) (connector.Connector, error)`
3. Implement appropriate connector interface(s) (`CallbackConnector`, `PasswordConnector`, etc.)
4. Register in `cmd/dex/config.go` in `ConnectorsConfig` map
5. Add configuration struct and validation

### Adding a New Storage Backend

1. Create package: `storage/mybackend/`
2. Implement `Config` struct with `Open(logger) (storage.Storage, error)`
3. Implement all `storage.Storage` interface methods
4. Register in `cmd/dex/config.go` in `storageConfigs` map
5. Run conformance tests from `storage/conformance/`

### Adding a gRPC API Method

1. Update `api/v2/api.proto` with new message types and RPC
2. Run `make generate-proto` to regenerate Go code
3. Implement method in `server/api.go` in `dexAPI` type
4. Increment `apiVersion` constant in `server/api.go`

## Testing Practices

- Unit tests use in-memory storage backend (`storage/memory`)
- Integration tests use `storage/conformance` suite to verify all backends
- Connector tests often use mock upstream providers
- Tests should clean up resources (auth codes, refresh tokens, etc.)
- Race detector is enabled in CI (`make testrace`)

## Code Style

Dex uses golangci-lint with these key linters:
- `gofmt`, `gofumpt`, `goimports`: Formatting
- `govet`: Go correctness checks
- `misspell`: Spelling (US locale)
- `unused`, `ineffassign`: Dead code detection
- `depguard`: Prevent deprecated imports (e.g., `io/ioutil`)

Run `make fix` to auto-format code.

## Security Considerations

- Client secrets and passwords are stored in storage backend (encrypted at rest if backend supports)
- Passwords use bcrypt with high cost (12-16)
- ConnectorData is opaque and never exposed to clients
- Refresh tokens are rotated on each use
- Keys are rotated periodically with old keys retained for JWT verification
- CORS is restricted by configuration
- TLS 1.2+ minimum version supported
