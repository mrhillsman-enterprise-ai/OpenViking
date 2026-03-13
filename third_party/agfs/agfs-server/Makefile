.PHONY: all build run test clean install help lint deps

# OS detection
ifeq ($(OS),Windows_NT)
    PLATFORM := windows
    SHELL_OS := windows
else
    UNAME_S := $(shell uname -s)
    ifeq ($(UNAME_S),Linux)
        PLATFORM := linux
        SHELL_OS := linux
    endif
    ifeq ($(UNAME_S),Darwin)
        PLATFORM := darwin
        SHELL_OS := darwin
    endif
endif

# Variables
BINARY_NAME=agfs-server
BUILD_DIR=build
CMD_DIR=cmd/server
GO=go
GOFLAGS=-v
ADDR?=:8080

# Platform-specific settings
ifeq ($(PLATFORM),windows)
    BINARY_EXT := .exe
    LIB_EXT := .dll
    MKDIR := if not exist $(BUILD_DIR) mkdir
    RM := del /Q /F
    CP := copy
    LIB_PREFIX :=
else
    BINARY_EXT :=
    MKDIR := mkdir -p
    RM := rm -rf
    CP := cp
    LIB_PREFIX := lib
    ifeq ($(PLATFORM),darwin)
        LIB_EXT := .dylib
    else
        LIB_EXT := .so
    endif
endif

# Build information
VERSION?=$(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
ifeq ($(PLATFORM),windows)
    # Use powershell for cross-platform date on windows if possible, or fallback
    BUILD_TIME := $(shell powershell -Command "Get-Date -uformat '%Y-%m-%d_%H:%M:%S'" 2>nul || echo "unknown")
else
    BUILD_TIME=$(shell date -u '+%Y-%m-%d_%H:%M:%S')
endif
GIT_COMMIT=$(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")

# LDFLAGS for build information
LDFLAGS=-ldflags "-X main.Version=$(VERSION) -X main.BuildTime=$(BUILD_TIME) -X main.GitCommit=$(GIT_COMMIT)"

all: build ## Build the project (default target)

build: ## Build the server binary
	@echo "Building $(BINARY_NAME) for $(PLATFORM)..."
	@$(MKDIR) $(BUILD_DIR)
	$(GO) build $(GOFLAGS) $(LDFLAGS) -o $(BUILD_DIR)/$(BINARY_NAME)$(BINARY_EXT) $(CMD_DIR)/main.go
	@echo "Build complete: $(BUILD_DIR)/$(BINARY_NAME)$(BINARY_EXT)"

build-lib: ## Build AGFS binding library
	@echo "Building binding library for $(PLATFORM)..."
	@$(MKDIR) $(BUILD_DIR)
ifeq ($(PLATFORM),darwin)
	CGO_ENABLED=1 $(GO) build -buildmode=c-shared -o $(BUILD_DIR)/$(LIB_PREFIX)agfsbinding$(LIB_EXT) cmd/pybinding/main.go
else ifeq ($(PLATFORM),linux)
	CGO_ENABLED=1 $(GO) build -buildmode=c-shared -o $(BUILD_DIR)/$(LIB_PREFIX)agfsbinding$(LIB_EXT) cmd/pybinding/main.go
else ifeq ($(PLATFORM),windows)
	@# Check if we are cross-compiling or building locally
	@if [ "$$(uname -s 2>/dev/null)" = "" ]; then \
		set CGO_ENABLED=1 && $(GO) build -buildmode=c-shared -o $(BUILD_DIR)/agfsbinding$(LIB_EXT) cmd/pybinding/main.go; \
	else \
		CC=x86_64-w64-mingw32-gcc CXX=x86_64-w64-mingw32-g++ AR=x86_64-w64-mingw32-ar GOOS=windows GOARCH=amd64 CGO_ENABLED=1 $(GO) build -buildmode=c-shared -o $(BUILD_DIR)/agfsbinding$(LIB_EXT) cmd/pybinding/main.go; \
	fi
endif
	@echo "Build complete in $(BUILD_DIR)"

run: build
	@echo "Starting $(BINARY_NAME) on $(ADDR)..."
	./$(BUILD_DIR)/$(BINARY_NAME)$(BINARY_EXT) -addr $(ADDR)

dev: ## Run the server in development mode (without building binary)
	@echo "Running server in development mode on $(ADDR)..."
	$(GO) run $(CMD_DIR)/main.go -addr $(ADDR)

install: build ## Install the binary to $GOPATH/bin
	@echo "Installing $(BINARY_NAME) to $(GOPATH)/bin..."
	$(GO) install $(LDFLAGS) $(CMD_DIR)/main.go
	@echo "Installed successfully"

test: ## Run all tests
	@echo "Running tests..."
	$(GO) test -v ./...

lint: ## Run golangci-lint (requires golangci-lint to be installed)
	@echo "Running golangci-lint..."
	@if command -v golangci-lint >/dev/null 2>&1; then \
		golangci-lint run ./...; \
	else \
		echo "golangci-lint not installed. Install it from https://golangci-lint.run/usage/install/"; \
		exit 1; \
	fi

deps: ## Download dependencies
	$(GO) mod download
	$(GO) mod tidy

deps-update: ## Update dependencies
	$(GO) get -u ./...
	$(GO) mod tidy

clean: ## Clean build artifacts
	@echo "Cleaning..."
	@$(RM) $(BUILD_DIR)
	@$(RM) coverage.out coverage.html
	@echo "Clean complete"

docker-build: ## Build Docker image
	@echo "Building Docker image..."
	docker build -t agfs-server:$(VERSION) .

docker-run: ## Run Docker container
	@echo "Running Docker container..."
	docker run -p 8080:8080 agfs-server:$(VERSION)

release: clean test build ## Run tests and build release binary
	@echo "Creating release build..."
	@$(MKDIR) $(BUILD_DIR)/release
	GOOS=linux GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BUILD_DIR)/release/$(BINARY_NAME)-linux-amd64 $(CMD_DIR)/main.go
	GOOS=linux GOARCH=arm64 $(GO) build $(LDFLAGS) -o $(BUILD_DIR)/release/$(BINARY_NAME)-linux-arm64 $(CMD_DIR)/main.go
	GOOS=darwin GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BUILD_DIR)/release/$(BINARY_NAME)-darwin-amd64 $(CMD_DIR)/main.go
	GOOS=darwin GOARCH=arm64 $(GO) build $(LDFLAGS) -o $(BUILD_DIR)/release/$(BINARY_NAME)-darwin-arm64 $(CMD_DIR)/main.go
	GOOS=windows GOARCH=amd64 $(GO) build $(LDFLAGS) -o $(BUILD_DIR)/release/$(BINARY_NAME)-windows-amd64.exe $(CMD_DIR)/main.go
	@echo "Release builds complete in $(BUILD_DIR)/release/"

help: ## Display this help screen
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
