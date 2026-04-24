# Title

* Project Drasi - 2026 Apr 24 - Ahmed Kamal (@ahmed-kamal2004)

## Overview

Shell Reaction is a Drasi Reaction component that executes operating system commands in response to continuous query result changes. This enables users to connect Drasi detections and state transitions to operational automations.

This design targets drasi-core reactions and aligns with the Drasi reaction model: Source -> Query -> Reaction.

## Terms and definitions

| Term | Definition |
|------|------------|
| Shell Reaction | A Drasi Reaction that executes configured system commands based on query change events |
| Reaction event | A single event emitted by a Query Container (insert, update, delete) |
| Execution policy | Configuration that constrains command path, arguments, environment vars, input data templates, timeout, and concurrency |

### User scenarios

- An operator runs continuous queries that detect abnormal conditions (for example sustained threshold violations). They configure Shell Reaction to invoke an operational script that creates a ticket, email or invokes internal/external endpoint.

- An application engineer wants to integrate Drasi with an existing internal tool that only exposes a CLI. Instead of building a new reaction service, they configure Shell Reaction to run the tool with event-derived arguments.

### Goals

1. Deliver a production-grade Secure Shell Reaction.
2. Support event-to-command mapping with explicit templates.
3. Provide runtime controls: timeout, retries, kill-on-drop and max concurrency.
4. Ensure integration with current Drasi components.
5. Testing coverage for unit and integration tests.

## Design requirements

### Requirements

1. Must run as a Drasi Reaction component and consume query result change events from Query Container.
2. Must validate configuration at startup and fail fast on invalid command templates or policy settings.
3. Must support both executable input modes:
	- stdin piped from parent process
	- environment variables
4. Must enforce the Single Binary Rule: `tokio::process::Command` executes one binary path from `command`.
5. Must validate executable on startup (exists, executable bit/permissions).
6. Must support non-blocking spawn execution.
7. Must support `max_concurrent` runners with bounded queue behavior.
8. Must support `kill_on_drop` option for spawned processes.
9. Must include unit tests and integration tests.

### Dependencies

- drasi-core reaction implementation patterns.
- Continuous query engine output mapping.
- Linux.

### Out of scope

## Design

### High-level design

Shell Reaction receives query events and executes one process per event. Execution is performed with non-blocking spawn and bounded concurrency.

Executable data ingress is supported in two modes:
1. `stdin` mode: event-derived payload is piped to child stdin.
2. `env` mode: selected event fields are projected to environment variables.

At startup, the reaction validates the configured executable path and enforces the Single Binary Rule.

### Architecture Diagram

```text
Query Container events
				|
				v
Shell Reaction
				v
Configured executable
	- stdin pipe from parent (optional)
	- env vars from event mapping (optional)
```

### Detail design

#### 1. Executable input modes

Shell Reaction supports both delivery channels for event data:
1. `stdin` mode: serialize selected payload template and pipe into child stdin.
2. `env` mode: map selected event fields to environment variables.

Both modes can be enabled together.

Example config per query.

Assume for query
```
    MATCH (rd:readings)
    WHERE rd.temp > 100
    RETURN rd.floor AS floor, rd.temp AS temp
```

```yaml
- query_name: "query_1"
  execute: "/bin/python3 /scripts/handle_event.py"
  mapping:
    insert:
      stdin: "[{{query_name}}] + {{after.floor}}: ${{after.temp}}"
      env:
        FLOOR: "{{after.floor}}"
        TEMP: "{{after.temp}}"
    update:
      stdin: "[{{query_name}}] ~ {{after.floor}}: ${{before.temp}} ${{after.temp}}"
      env:
        FLOOR: "{{after.floor}}"
        TEMP: "{{after.temp}}"
    delete:
      env:
        FLOOR: "{{before.floor}}"
        TEMP: "{{before.temp}}"
- // other queries
```

or (depending on discussion) (second option is more flexible, follows the Separation of Responsibilities principle)

```yaml
- query_name: "query_1"
  mapping:
    insert:
      execute: "/bin/python3 /scripts/handle_event_insert.py"
      stdin: "[{{query_name}}] + {{after.floor}}: ${{after.temp}}"
      env:
        FLOOR: "{{after.floor}}"
        TEMP: "{{after.temp}}"
    update:
      execute: "/bin/python3 /scripts/handle_event_update.py"
      stdin: "[{{query_name}}] ~ {{after.floor}}: ${{before.temp}} ${{after.temp}}"
      env:
        FLOOR: "{{after.floor}}"
        TEMP: "{{after.temp}}"
        BEFORE_TEMP: "{{before.temp}}"
    delete:
      execute: "/bin/python3 /scripts/handle_event_delete.py"
      stdin: "[{{query_name}}] - {{before.floor}}: ${{before.temp}}"
- // other queries
```

Note: both of `stdin` and `env` are optional per event type, but at least one must be provided.

Last-mile security problem:
It depends on how the executable processes the input data, which can't be fully controlled by drasi, especially if there is bad input inserted to the drasi query engine, and then has been processed by the executable (this is issue can be mitigated by seccomp integration and proper documentation for best practices).


#### 2. Single Binary Rule

`tokio::process::Command` executes a single binary configured in `Command::new()`.

Shell Words crate can be used for parsing (splitting) the command `execute` string into executable and args

Example:
```rust
let command_str = "/bin/python3 /scripts/handle_event.py --option value";
let parts = shell_words::split(command_str)?;
let executable = parts[0]; // "/bin/python3"
let args = &parts[1..]; // ["/scripts/handle_event.py", "--option", "value"]
```

Startup validation checks for the executable:
1. path exists
2. path points to a file, not directory
3. executable permission is set

#### 3. Runtime model
- `status` as blocking running/completed/failed with exit code and error details.

- `spawn` as non-blocking and tracks in-flight children.

Example flow:
```rust
let mut child = Command::new(executable)
    .args(args)
    .envs(env_mapping) // set environment variables from mapping
    .stdin(Stdio::piped())
    .spawn()?;

let mut stdin = child.stdin.take().expect("Failed to open stdin");

tokio::spawn(async move {
    // write event data to stdin
    stdin.write(event_mapped_payload.as_bytes()).await.expect("Failed to write to stdin");
    drop(stdin);
});

// wait for exit
let _ = child.wait().await.expect("Failed to wait for child process");

// [optional] get exit status and output.


```

Controls:
1. `max_concurrent` limits active child processes.
2. queue/backpressure policy is applied when at capacity.
3. `kill_on_drop` can be enabled so process is terminated when handle is dropped during shutdown/restart.

##### Failure retries:
1. if child process exits with non-zero code, classify as failure and apply retry policy if configured.

`retry_on_failure` example config:
```yaml
execute: "/bin/python3 /scripts/handle_event.py"
retry_on_failure:
  enabled: true
  max_retries: 3
  base_delay_ms: 1000 ## ( ms base ) exponential backoff with 2 multiplier
```

#### 4. Seccomp implementation proposal

This section follows the intended Rust model using `tokio::process::Command`, `std::os::unix::process::CommandExt::pre_exec`, and `seccompiler::apply_filter`.

Design behavior:
1. Build a seccomp filter before spawn using a selected set of capabilities or profile.
2. Apply the filter in `pre_exec` (child process after fork, before exec).

##### Models
- Capabilities model:
Giving capabilities based on configuration, for example: `read_files`, `write_files`, `network`, `delete_files` and so on.

- Profile model:
common used capabilities can be grouped into profiles together. For example, `logger` profile can include `write_files` capability, and `changer` profile can include `write_files` and `delete_files` capabilities.

Config shape (proposal):

```yaml
execute: "/bin/python3 /scripts/handle_event.py"
hardening:
  seccomp:
    enabled: true
    capabilities:
      - read_files
      - write_files
  process:
    envClear: true
    minimalPath: "/usr/local/bin:/usr/bin"
```

Implementation notes:
1. This feature is only present in Linux.

#### 5. Env variable clearance
To minimize inherited environment variable risks, we propose an `env_clear` option that starts with an empty environment for the child process. The reaction configuration can then specify a set of needed variables from parent environment within the execution context.

example config:
```yaml
execute: "/bin/python3 /scripts/handle_event.py"
hardening:
  process:
    env_clear: true
    envs: [PATH, CUSTOM_VAR]
        
```

### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable.

- Describe the REST APIs in detail for new resource types or updates to existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components. -->

full config example:

```yaml
max_concurrent: 5
kill_on_drop: true
retry_on_failure:
  max_retries: 3
  base_delay_ms: 1000
queries:
  - query_name: "query_1"
    execute: "/bin/python3 /scripts/handle_event.py"
    hardening:
      seccomp:
        enabled: true
        capabilities:
          - read_files
          - write_files
      env_clear: true
      envs: [PATH, CUSTOM_VAR]
    mapping:
        insert:
          stdin: "[{{query_name}}] + {{after.floor}}: ${{after.temp}}"
          env:
              FLOOR: "{{after.floor}}"
              TEMP: "{{after.temp}}"
        update:
          stdin: "[{{query_name}}] ~ {{after.floor}}: ${{before.temp}} ${{after.temp}}"
          env:
              FLOOR: "{{after.floor}}"
              TEMP: "{{after.temp}}"
              BEFORE_TEMP: "{{before.temp}}"
        delete:
          stdin: "[{{query_name}}] - {{before.floor}}: ${{before.temp}}"
          env:
              FLOOR: "{{before.floor}}"
              TEMP: "{{before.temp}}"

### Alternatives Considered

<!-- Describe the alternative designs that were considered or should be considered. Give a justification for why alternative approaches should be rejected if possible. -->

## Security

Security model in this design prioritizes constrained execution:

1. Single executable path with startup validation reduces command injection surface.
2. stdin/env mappings are explicit and template-driven.
3. Non-root execution is required. (to be discussed)
4. `kill_on_drop` helps optional avoidance of orphaned processes on crash/redeploy paths.

Additional hardening options (proposal):
1. seccomp profile and Linux capability reduction
2. `env_clear` to start with a minimal environment.

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

## Supportability

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->

### Verification

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

## Development Plan

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

## Open issues
<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design. -->

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

## References

<!-- Optional. Add the design documents and references you use for this document. -->
- https://docs.rs/shell-words/1.1.1/shell_words/
- https://www.stackhawk.com/blog/rust-command-injection-examples-and-prevention/
- https://ssojet.com/escaping/shell-escaping-in-rust#safe-command-execution-in-rust
- https://mojoauth.com/escaping/shell-escaping-in-rust#real-world-examples-for-shell-escaping-using-rust
- https://docs.rs/seccompiler/latest/seccompiler/