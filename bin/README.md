
# Plane Prime CLI

A Go CLI tool that installs, orchestrates, upgrades, and uninstalls Plane services on client machines. It communicates with a Prime server API, manages Docker containers via compose, and provides an interactive TUI.

## Build & Dev Commands

```bash
# Build the binary
go build ./cmd/prime-cli

# Build with version info (how CI does it)
go build -ldflags "-s -w -X main.CT_VERSION=v1.0.0 -X main.CT_COMMIT=abc1234" ./cmd/prime-cli

# Vet all packages
go vet ./...

# Run all tests
go test ./...

# Tidy dependencies
go mod tidy
```

Release builds use GoReleaser (`.goreleaser.yaml`), triggered via GitHub Actions.

## Project Structure

Every CLI command follows: **Cobra command → Handler → TUI model (Bubble Tea)**

1. **`internal/cmd/`** — Cobra command definitions. Each command injects dependencies and delegates to a handler. Commands use `RunE` (not `Run`) for error propagation.
2. **`internal/handlers/`** — Each command has a handler implementing a `Run()` method. Handlers wire up services and create TUI models.
3. **`internal/tui/`** — Bubble Tea models for interactive UI. Each operation (setup, start, stop, upgrade, etc.) has its own model file plus a `*_stages.go` file defining the multi-step stage progression.

```
.
├── cmd/
│   └── prime-cli/
│       └── main.go
├── internal/
│   ├── api/                  # HTTP client for the Prime server API (IPrimeApi interface)
│   ├── cmd/                  # Cobra command definitions
│   │   ├── backup.go
│   │   ├── configure.go
│   │   ├── healthcheck.go
│   │   ├── monitor.go
│   │   ├── pull.go
│   │   ├── repair.go
│   │   ├── restart.go
│   │   ├── restore.go
│   │   ├── root.go
│   │   ├── setup.go
│   │   ├── ssl.go
│   │   ├── start.go
│   │   ├── stop.go
│   │   ├── uninstall.go
│   │   ├── update.go
│   │   ├── upgrade.go
│   │   └── validate.go
│   ├── constants/            # Base paths, Docker image names, logos, env var definitions
│   ├── descriptors/          # Command descriptions, regex patterns, TUI display messages
│   ├── docker/               # Docker SDK wrapper (compose up/down, image pulls, containers)
│   ├── environment/          # Manages env files (plane.env, variables.env, .config.env)
│   ├── errors/               # Error message constants
│   ├── handlers/             # Command handlers (backup, restore, setup, start, stop, upgrade, validate)
│   ├── logger/               # Event logger (sends structured events to Prime API)
│   ├── messages/             # Bubble Tea message types shared across TUI models
│   ├── services/             # Config loading, file validation, file downloading
│   ├── tui/                  # Bubble Tea TUI models and views
│   │   ├── app_stages.go
│   │   ├── backup.go
│   │   ├── configure.go
│   │   ├── configure_view.go
│   │   ├── enter_comp.go
│   │   ├── health_check.go
│   │   ├── logger.go
│   │   ├── monitor.go
│   │   ├── pull.go
│   │   ├── question.go
│   │   ├── question_suite.go
│   │   ├── repair.go
│   │   ├── repair_stages.go
│   │   ├── restore.go
│   │   ├── setup.go
│   │   ├── setup_stages.go
│   │   ├── splash_screen.go
│   │   ├── start.go
│   │   ├── stop.go
│   │   ├── styles.go
│   │   ├── types/
│   │   ├── uninstall.go
│   │   ├── uninstall_stages.go
│   │   ├── upgrade.go
│   │   ├── upgrade_stages.go
│   │   └── views/
│   └── utils/                # File I/O, backup/restore (tar.gz), cron jobs, CLI version checks
├── go.mod
├── go.sum
├── .goreleaser.yaml
└── CLAUDE.md
```

### Key Interfaces

```go
type IPrimeApi interface {
    Setup(CredentialsPayload) (SetupResponse, error)
    TransmitStatus(StatusRequest) error
    FetchLatestVersion(PlaneVersion) (LatestVersionResponse, error)
    GetConfigurationFilesURL(ConfigurationPayload) (AssetsPayloadResponse, error)
    DownloadPrimeCLI(EnvironmentPayload) ([]byte, error)
    UpgradeVersion(UpgradeVersionPayload) error
    ValidateInstance(InstanceValidationPayload) error
    LogEvent(AppLogPayload) error
    GetLatestCLIVersion() (LatestCLIVersionResponse, error)
    ActivateInstance() error
}
```

### Package Name Aliases

Several packages use custom names in imports — maintain these conventions:
- `DockerSDK` for `internal/docker`
- `EventLogger` for `internal/logger`
- `error_msgs` for `internal/errors`
- `global_types` for `internal/tui/types`
- `global_views` for `internal/tui/views/global`
- `upgrade_view` for `internal/tui/views/upgrade`

### TUI Stage Pattern

Multi-step operations (setup, upgrade, repair, uninstall) use a stages pattern:
- `*_stages.go` files define ordered `[]StageMessage` and `[]func(AssetStruct) func() tea.Msg`
- The TUI model advances through stages sequentially, each stage function returning a `tea.Msg`
- `AssetStruct` carries shared state (host, platform, API client, Docker service) across stages

### Version Injection

`CT_VERSION` and `CT_COMMIT` are set via `-ldflags` at build time. At runtime, version is `"development"` unless overridden.

## Tests

Tests cover the core packages:

| Package | Coverage |
|---------|----------|
| `internal/utils` | Env file parsing, tar/gz, stage messages, progress reader, IP classification |
| `internal/services` | Config load/save, environment validation, IP address detection |
| `internal/tui` | URL validators (Redis, Postgres, AMQP), AWS/S3/email validation, question suite construction |
| `internal/constants` | Image name generation, env var map, default values |
| `internal/api` | API client construction, HTTP response handling, endpoint constants, mock server tests |

Run tests with:

```bash
go test -vet=off ./internal/utils/ ./internal/services/ ./internal/tui/ ./internal/constants/ ./internal/api/ -v
```

> **Note:** Some packages have pre-existing `go vet` warnings (`non-constant format string in fmt.Errorf`), so `-vet=off` is needed until those are addressed.

## Commands

**Installation & Maintenance**

- **`setup`** — Install Docker if necessary, download configuration files and Docker images, start the Plane app.
- **`upgrade`** — Upgrade the Plane app to the latest available version.
- **`repair`** — Repair an existing installation.
- **`uninstall`** — Remove configuration files, Docker images, and gracefully shut down services.
- **`update-cli`** — Update the CLI itself to the latest version.
- **`validate`** — Validate the current setup.

**Orchestration**

- **`start`** — Start Plane services.
- **`stop`** — Stop Plane services.
- **`restart`** — Restart Plane services.
- **`pull`** — Pull the latest images for the current version.

**Configuration & Monitoring**

- **`configure`** — Configure Plane app settings (storage provider, external DB, ports, etc.).
- **`ssl`** — Configure SSL for the instance.
- **`monitor`** — Display logs and status of Plane services.
- **`healthcheck`** — Check the health of all Plane services.

**Backup & Restore**

- **`backup`** — Back up the entire installation along with data.
- **`restore`** — Restore the installation from a selected backup.

## Environment Variables

| Variable | Description | Default |
| :------- | :---------- | :------ |
| `PRIME_MIGRATION_TIMEOUT` | Maximum time the CLI waits for database migrations to complete during `setup`/`upgrade` before timing out. Accepts a Go duration (e.g. `90m`, `1h30m`) or a plain integer treated as minutes (e.g. `90`). Increase this for large instances whose fresh migrations run long. | `60m` |

Example — allow up to 90 minutes for migrations on a large instance:

```bash
PRIME_MIGRATION_TIMEOUT=90m prime-cli setup
```
