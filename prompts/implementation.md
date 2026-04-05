# AI Implementation Prompts

## Purpose

This document provides prompts to instruct an AI model to generate the CarrySprint implementation.  
Each prompt is self-contained.  
Each prompt references the specification documents in this repository.

---

## Prompt 1 — Local Server (Go)

```
You are implementing the CarrySprint local server in Go.

Read the following specification documents:
- architecture/overview.md
- api/spec.md
- database/schema.md
- operations/model.md
- constraints/rules.md

Requirements:
1. Create a Go module named `carrysprint-server`.
2. Use the standard library `net/http` for the HTTP server.
3. Use `github.com/mattn/go-sqlite3` for SQLite access.
4. Use `github.com/BurntSushi/toml` for configuration parsing.
5. Implement all API endpoints defined in api/spec.md.
6. Implement all database tables defined in database/schema.md.
7. Implement all constraints defined in constraints/rules.md.
8. Follow the startup and shutdown sequence in operations/model.md.
9. Write structured JSON logs to ~/.carrysprint/server.log.
10. Listen on 127.0.0.1:7654 by default.
11. Reject non-loopback connections with HTTP 403.
12. Store the PID in ~/.carrysprint/server.pid on startup.
13. Remove the PID file on graceful shutdown.
14. Apply database migrations automatically on startup.

Output only the Go source files required to implement the server.
Do not include explanation or commentary outside of Go code comments.
```

---

## Prompt 2 — CLI Client (Go)

```
You are implementing the CarrySprint CLI client in Go.

Read the following specification documents:
- architecture/overview.md
- api/spec.md
- operations/model.md

Requirements:
1. Create a Go module named `carrysprint-cli`.
2. The binary must be named `cs`.
3. Use `github.com/spf13/cobra` for command parsing.
4. Implement all commands listed in operations/model.md under "CLI Commands".
5. Read the server host and port from ~/.carrysprint/config.toml.
6. Default to http://127.0.0.1:7654 if config.toml is missing.
7. Send HTTP requests to the local server for all data operations.
8. Parse JSON responses from the server.
9. Format and print output as human-readable text.
10. Exit with code 0 on success.
11. Exit with code 1 on error.
12. Print error messages to stderr.

Output only the Go source files required to implement the CLI.
Do not include explanation or commentary outside of Go code comments.
```

---

## Prompt 3 — Database Migrations

```
You are implementing the SQLite database migration system for CarrySprint.

Read the following specification documents:
- database/schema.md
- operations/model.md

Requirements:
1. Implement a migration runner in Go.
2. Migrations are numbered starting from 0001.
3. Each migration is a .sql file embedded in the binary.
4. Applied migrations are tracked in the schema_migrations table.
5. The runner applies only migrations that have not been applied yet.
6. The runner runs automatically during server startup.
7. Write the initial migration (0001) that creates the sprints and tasks tables.
8. Write the indexes defined in database/schema.md.

Output only the Go source files and SQL migration files required.
Do not include explanation or commentary outside of code comments.
```

---

## Prompt 4 — Carry Operation

```
You are implementing the carry operation for CarrySprint.

Read the following specification documents:
- api/spec.md (section: Carry)
- constraints/rules.md (section: Carry Rules)
- database/schema.md

Requirements:
1. Implement the POST /sprints/:id/carry endpoint in Go.
2. Validate all Carry Rules from constraints/rules.md (CR-01 through CR-06).
3. Copy only tasks with status != "done" from the source sprint to the target sprint.
4. Each copied task is a new database row with a new auto-incremented ID.
5. Set status = "todo" on all copied tasks.
6. Return the count of carried tasks in the response body as {"carried": N}.
7. Run the entire operation in a single database transaction.
8. Roll back the transaction on any error.

Output only the Go source code for this endpoint handler.
Do not include explanation or commentary outside of Go code comments.
```

---

## Prompt 5 — Unit Tests

```
You are writing unit tests for the CarrySprint local server in Go.

Read the following specification documents:
- api/spec.md
- constraints/rules.md

Requirements:
1. Write tests using the standard library `testing` package.
2. Use an in-memory SQLite database for tests.
3. Write at least one test per API endpoint.
4. Write tests that validate all constraint rules in constraints/rules.md.
5. Test all error responses (404, 422, 500).
6. Test the carry operation for all Carry Rules.
7. Test that the server rejects non-loopback requests with HTTP 403.

Output only Go test files.
Do not include explanation or commentary outside of Go code comments.
```

---

## Prompt 6 — Configuration and Logging

```
You are implementing configuration loading and structured logging for CarrySprint.

Read the following specification documents:
- architecture/overview.md
- operations/model.md

Requirements:
1. Load configuration from ~/.carrysprint/config.toml using BurntSushi/toml.
2. Create the file with default values if it does not exist.
3. Support these keys: server.host, server.port, server.log_level, database.path.
4. Implement structured JSON logging to ~/.carrysprint/server.log.
5. Each log line must be a single JSON object with fields: time, level, msg.
6. Support log levels: debug, info, warn, error.
7. Default log level is info.
8. Apply the configured log level to filter output.

Output only the Go source files for configuration and logging.
Do not include explanation or commentary outside of Go code comments.
```
