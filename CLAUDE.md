# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Immich-go is a Go CLI application for uploading photos to Immich servers. It provides an alternative to the official Immich CLI that doesn't require Node.js. It supports multiple sources including Google Photos Takeouts, iCloud, local folders, and server-to-server migrations.

## Build Commands

```bash
# Build the binary
go build -o immich-go main.go

# Build for specific platforms
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o immich-go-linux-amd64 -ldflags="-s -w -extldflags=-static" main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -o immich-go-windows-amd64.exe -ldflags="-s -w -extldflags=-static" main.go
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -o immich-go-darwin-amd64 -ldflags="-s -w -extldflags=-static" main.go
CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -o immich-go-darwin-arm64 -ldflags="-s -w -extldflags=-static" main.go

# Install dependencies
go mod download
go mod tidy
```

## Test Commands

```bash
# Run all tests
go test ./...

# Run tests with race detection
go test -race -v -count=1 ./...

# Run a specific test
go test -v -run TestFunctionName ./package/...

# Run tests for a specific package
go test -v ./immich/...

# Run security vulnerability scan
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

## Lint Commands

```bash
# Run golangci-lint (requires installation)
golangci-lint run

# Run with extended timeout
golangci-lint run --timeout=10m
```

## Running the Application

```bash
# Upload from a local folder
./immich-go upload from-folder --server=http://your-ip:2283 --api-key=your-api-key /path/to/photos

# Upload Google Photos takeout
./immich-go upload from-google-photos --server=http://your-ip:2283 --api-key=your-api-key /path/to/takeout-*.zip

# Archive from Immich server
./immich-go archive from-immich --from-server=http://your-ip:2283 --from-api-key=your-api-key --write-to-folder=/path/to/archive

# Dry run (simulates without uploading)
./immich-go upload from-folder --dry-run --server=http://your-ip:2283 --api-key=your-api-key /path/to/photos
```

## Architecture Overview

### Command Structure

The application uses the Cobra CLI framework with a command hierarchy:

- `app/root/rootCmd.go` - Root command setup, initializes the Application context
- `app/upload/` - Upload command and subcommands (from-folder, from-google-photos, from-immich, etc.)
- `app/archive/` - Archive command for downloading from Immich
- `app/stack/` - Stack command for managing photo stacks
- `app/version/` - Version command

### Core Components

**Application Context** (`app/app.go`):
- Central configuration holder shared across all commands
- Manages logging, concurrency limits, dry-run mode, error handling
- Contains the FileProcessor for coordinated asset tracking

**Immich Client** (`immich/`):
- `ImmichClient` struct wraps the HTTP client for API communication
- Interface-based design (`ImmichInterface`) allows mocking for tests
- Handles retries, SSL configuration, and API tracing
- Key interfaces: `ImmichAssetInterface`, `ImmichAlbumInterface`, `ImmichTagInterface`, `ImmichStackInterface`

**Asset Model** (`internal/assets/`):
- `Asset` struct represents a file to be processed/uploaded
- Contains metadata: capture date, GPS coordinates, albums, tags, favorites
- Handles checksum computation for duplicate detection
- Supports sidecar metadata (XMP, JSON)

**Adapters** (`adapters/`):
- Reader pattern for different asset sources
- `adapters/folder/` - Local folder reading with Picasa/iCloud support
- `adapters/googlePhotos/` - Google Photos Takeout processing
- `adapters/fromimmich/` - Server-to-server migration
- `adapters/adapters.go` - Common `Reader` and `AssetWriter` interfaces

**File Processing** (`internal/fileprocessor/`):
- Coordinates asset tracking and event logging
- Manages duplicate detection
- Handles batching and upload queuing

### Configuration System

**Configuration Manager** (`internal/config/`):
- Supports YAML configuration files (default: `immich-go.yaml`)
- Command-specific configuration sections
- Environment variable and flag binding via Viper
- `--save-config` flag persists current settings

### Test Architecture

**Unit Tests**:
- Standard Go testing with `testify` for assertions
- HTTP mocks using `httptest` for Immich API calls
- Tests use external test packages (`package immich_test`)

**E2E Tests** (`internal/e2e/`):
- Requires running Immich server (via Docker)
- Scripts in `scripts/e2e/` for provisioning and cleanup
- Tests real upload/download scenarios
- CI runs E2E tests separately with maintainer approval for external PRs

### Branching Model

- `main` - Stable release branch
- `develop` - Integration branch for features
- `feature/*` and `bugfix/*` branches based on and merged into `develop`
- `hotfix/*` branches based on and merged into `main`

### Release Process

1. Generate release notes: `./scripts/generate-release-notes.sh v0.x.x`
2. Process the prompt file to generate final release notes
3. Create PR from develop to main named `release: [version]`
4. Tag the release after merge

## Key Patterns

### Error Handling
- `--on-errors` flag controls behavior: `stop`, `continue`, or numeric tolerance
- Context cancellation handled gracefully for Ctrl+C
- `ProcessError()` method on Application counts and handles errors

### Concurrency
- `--concurrent-tasks` controls parallelism (1-20, default: CPU count)
- Clamped to safe ranges in `PersistentPreRunE`
- Used for parallel uploads and processing

### API Implementation
When implementing Immich API endpoints:
1. Check API specs at `.github/immich-api-monitor/immich-openapi-specs-baseline.json`
2. Add method to `ImmichClient` in appropriate file (asset.go, album.go, etc.)
3. Add to the interface for mocking capability
4. Follow existing retry and error handling patterns

## Development Notes

### Important File Locations
- API client: `immich/*.go`
- Asset model: `internal/assets/asset.go`
- File type support: `internal/filetypes/`
- CLI flags: `internal/cliFlags/`
- E2E tests: `internal/e2e/`

### Documentation
- User docs: `docs/` directory with comprehensive guides
- Release notes: `docs/releases/`
- This project maintains extensive documentation for users
