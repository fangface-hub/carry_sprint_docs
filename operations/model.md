# Operational Model

## Deployment Target

CarrySprint runs entirely on the user's local machine.  
No cloud infrastructure is required.  
No internet access is required during operation.

## Supported Operating Systems

- Linux (amd64, arm64)
- macOS (amd64, arm64)
- Windows (amd64)

## Server Process

The local server runs as a background process.  
The server writes its PID to `~/.carrysprint/server.pid` on startup.  
The server removes `server.pid` on graceful shutdown.  
The server listens on `127.0.0.1:7654` by default.

## Server Startup Sequence

1. Read configuration from `~/.carrysprint/config.toml`.
2. Apply defaults for any missing configuration keys.
3. Open the SQLite database at `~/.carrysprint/data.db`.
4. Run pending database migrations.
5. Write PID to `~/.carrysprint/server.pid`.
6. Start the HTTP listener.
7. Log `server started` to `~/.carrysprint/server.log`.

## Server Shutdown Sequence

1. Stop accepting new HTTP connections.
2. Wait up to 5 seconds for in-flight requests to complete.
3. Close the database connection.
4. Remove `~/.carrysprint/server.pid`.
5. Log `server stopped` to `~/.carrysprint/server.log`.

## CLI Invocation

The CLI client is invoked as a single binary named `cs`.  
Each CLI invocation starts, executes one command, and exits.  
The CLI client reads the server address from `~/.carrysprint/config.toml`.  
The CLI client exits with code `0` on success.  
The CLI client exits with code `1` on error.

## CLI Commands

| Command | Description |
|---|---|
| `cs server start` | Start the server as a background daemon |
| `cs server stop` | Stop the running server gracefully |
| `cs server status` | Print the server PID and status |
| `cs sprint list` | List all sprints |
| `cs sprint create` | Create a new sprint (interactive or via flags) |
| `cs sprint show <id>` | Show a sprint and its tasks |
| `cs sprint delete <id>` | Delete a sprint and all its tasks |
| `cs sprint carry <id>` | Carry incomplete tasks to another sprint |
| `cs task list <sprint_id>` | List tasks in a sprint |
| `cs task create <sprint_id>` | Create a task in a sprint |
| `cs task update <id>` | Update a task |
| `cs task delete <id>` | Delete a task |

## Logging

The server logs to `~/.carrysprint/server.log`.  
Each log line is a single JSON object.  
Each log line contains the fields: `time`, `level`, `msg`.  
The default log level is `info`.  
The log level is configurable via `config.toml`.

### Log Levels

| Level | Description |
|---|---|
| `debug` | Detailed internal events |
| `info` | Normal operational events |
| `warn` | Unexpected but recoverable events |
| `error` | Failures that affect a single request |

## Configuration File

The configuration file is located at `~/.carrysprint/config.toml`.  
The configuration file is created with default values if it does not exist.

```toml
[server]
host = "127.0.0.1"
port = 7654
log_level = "info"

[database]
path = "~/.carrysprint/data.db"
```

## Upgrade

A new binary replaces the old binary.  
The server applies any pending migrations automatically on the next startup.  
No manual migration steps are required.

## Data Backup

The database is a single SQLite file at `~/.carrysprint/data.db`.  
Users back up their data by copying that file.  
No built-in backup tooling is provided.
