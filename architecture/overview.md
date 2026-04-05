# Architecture Overview

## System Summary

CarrySprint is a local-first sprint and task management system.  
All data is stored and processed on the user's machine.  
No external network connection is required during normal operation.

## Components

### Local Server

The local server is written in Go.  
The local server runs as a background process on the user's machine.  
The local server listens on `127.0.0.1` only.  
The local server exposes a REST API over HTTP.  
The local server manages all persistence via a local SQLite database.

### CLI Client

The CLI client is a command-line application.  
The CLI client communicates with the local server via HTTP.  
The CLI client does not access the database directly.  
The CLI client formats and displays responses from the server.

## Component Diagram

```
+------------------+          HTTP/REST          +------------------+
|   CLI Client     | --------------------------> |   Local Server   |
|                  | <-------------------------- |   (Go process)   |
+------------------+       JSON responses        +--------+---------+
                                                          |
                                                          | SQL
                                                          v
                                                 +------------------+
                                                 |  SQLite Database |
                                                 |  (local file)    |
                                                 +------------------+
```

## Data Flow

1. The user invokes a CLI command.
2. The CLI client sends an HTTP request to the local server.
3. The local server processes the request and reads or writes the SQLite database.
4. The local server returns a JSON response.
5. The CLI client formats and prints the response.

## Process Lifecycle

The local server starts as a background daemon.  
The CLI client starts on demand and exits after each command.  
The local server writes a PID file to allow the CLI client to check its status.

## File Layout (runtime)

```
~/.carrysprint/
  server.pid       # PID of the running server process
  data.db          # SQLite database
  config.toml      # User configuration
  server.log       # Server log output
```

## Configuration

The server reads configuration from `~/.carrysprint/config.toml`.  
The server falls back to built-in defaults for any missing configuration key.  
The CLI client reads the server address and port from the same configuration file.

## Port

The default server port is `7654`.  
The port is configurable via `config.toml`.

## Security

The server listens only on the loopback interface (`127.0.0.1`).  
No authentication token is required for localhost-only requests.  
The server rejects all requests from non-loopback origins.
