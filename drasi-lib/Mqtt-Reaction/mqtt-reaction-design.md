# MQTT Reaction Design

* Project Drasi - April 7, 2026 - Ahmed Kamal (@ahmed-kamal)

## Overview

The MQTT Reaction in Drasi publishes continuous query result changes to MQTT brokers so external systems can consume Drasi output using a lightweight pub/sub protocol. This enables integration with IoT systems, edge gateways, automation platforms, and event-driven services that already rely on MQTT topics.

This design defines a production-ready MQTT Reaction built on the Drasi Reaction SDK. It focuses on reliable publish behavior, predictable topic and payload mapping and secure authentication and transport.

## Terms and definitions

| Term | Definition |
|------|------------|
| MQTT | Message Queuing Telemetry Transport protocol used for lightweight pub/sub messaging |
| Broker | MQTT server endpoint that receives published messages and routes to subscribers |
| Topic | Hierarchical routing key used by MQTT for message distribution |
| QoS | MQTT Quality of Service level (0, 1, 2) controlling delivery guarantees |
| Retain | MQTT flag that keeps the last message on a topic for new subscribers |

## Objectives

### User scenarios

**IoT Engineer Publishing Query Results to IoT Consumers**

An IoT engineer deploys a Drasi continuous query and wants every result change to be published to MQTT so downstream IoT devices and services can react immediately. They need to:
- Configure broker connectivity and authentication
- Control topic naming by query and event type
- Choose delivery guarantees (QoS/retain/inflight messages..etc)

### Goals

1. Add an MQTT Reaction for drasi-core that subscribes to queries and emits result changes to MQTT brokers.
2. Support MQTT v5 and v3.1.1 (starts with v5 and falls back to v3.1.1 if v5 is not supported on the broker).
3. Support per-query configuration of topic, payload template, QoS and retain settings.
5. TLS/SSL transport support and mTLS authentication support.
6. Support for Identity providers-based authentication (e.g. AWS IoT, Azure IoT, custom providers).
7. Integration with the Adaptive batching framework.
8. Validate interoperability with target brokers (Mosquitto).
9. Implement a testing suite covering unit tests and integration tests with a real broker.

### Non-Goals

## Design requirements

### Requirements

- Reaction implementation should align with patterns followed by other drasi-core reactions.
- Payload decoding must support JSON object payloads, basic scalar-in-object payloads or any other body format.
- Reaction implementation must support mqtt-reaction-global configuration.
- Reaction implementation must support configuration per query and operation.

### Dependencies

- drasi-core reaction implementation patterns.
- Broker availability and protocol support in test environments (Mosquitto).

### Out of scope

## Design

### High-level design

The MQTT Reaction receives query result events from the Drasi Query and transforms each event into a publish request composed of:
- target MQTT topic
- payload
- MQTT publish options (QoS, retain)

The reaction maintains a broker connection with reconnect logic (relying on rumqttc `AsyncClient`). Events are  processed then published to the broker.

### Architecture Diagram

```text
    Drasi Query Container
				|
				| result change events (insert/update/delete)
				v
	    MQTT Reaction
				|
				| MQTT publish
				v
		MQTT Broker
				|
				v
	Subscriber Applications
```

### Detail design

#### Startup Validation

##### Configuration Validation
At startup, the MQTT Reaction validates that for each query in `queries`, there is either a default configuration or a per-query configuration in `query_configs`. If any query lacks both, the reaction fails to start with a clear error message indicating the missing configuration. (Error)

Also the reaction validates that each query config in `query_configs` is targetting a query that is included in the `queries` list. If any query config targets a non-existent query, the reaction fails to start with an error indicating the invalid configuration. (Error)

##### Mqtt-related Configuration Validation
The reaction validates that all MQTT-related configuration fields are within acceptable ranges and formats. For example:
- `keep_alive` must be a positive integer greater than 5.
- `topic` must be a non-empty string.

The validation logic checks both global defaults and per-query overrides to ensure all effective configurations are valid. If any invalid configuration is detected, the reaction fails to start with a descriptive error message (Error).

#### Per Query Configuration

Relying on the current existing `QueryConfig` struct and `TemplateSpec` struct, we will introduce a new mqtt extension struct `MqttExtension`, which will include the MQTT specific configuration for each query. This struct will be added as an extension field in `TemplateSpec<MqttExtension>` to allow for MQTT-specific settings.

Introduction of new struct `MqttExtension` that includes the target topic name `topic`, the message template `template`, retaining method `retain` and the Quality of Service `qos`.


```rust
pub struct MqttExtension {
    /// Target Mqtt topic.
    pub topic: String,

    /// MQTT message retain policy.
    #[serde(default)]
    pub retain: Option<bool>, // default is no retain

    /// QoS level for MQTT messages.
    #[serde(default)]
    pub qos: Option<MqttQoS>, // default is QoS 1
}
```

The payload template is defined in the `template` field of the `TemplateSpec` struct.

Example for using `QueryConfig` and `MqttExtension` within the Mqtt Reaction:

```rust
let spec_add = TemplateSpec<MqttExtension> {
    template: "[{{query_name}}] + {{after.floor}}: {{after.temp}}".to_string(),
    extension: MqttExtension {
        topic: "example/topic".to_string(),
        retain: None,
        qos: Some(MqttQoS::AtMostOnce),
    },
};

let spec_update = TemplateSpec<MqttExtension> {
    template: "[{{query_name}}] ~ {{after.floor}}: {{before.temp}} {{after.temp}}".to_string(),
    extension: MqttExtension {
        topic: "example/topic".to_string(),
        retain: None,
        qos: Some(MqttQoS::AtMostOnce),
    },
};

let spec_delete = TemplateSpec<MqttExtension> {
    template: "DUMMY".to_string(),
    extension: MqttExtension {
        topic: "example/topic".to_string(),
        retain: None,
        qos: Some(MqttQoS::AtMostOnce),
    },
};

let query_config = QueryConfig {
    added: Some(spec_add),
    updated: Some(spec_update),
    deleted: Some(spec_delete),
};

// then insert the `query_config` to the MQTT reaction config routes map with the query id as a key.
```
If no per-query configuration is provided for a query, the reaction will fall back to a default configuration `default_template` defined in `MqttReactionConfig` for the reaction.

If no `TemplateSpec` is provided for a specific operation type (e.g. `added`, `updated`, `deleted`) in the query config, the reaction will fall back to the default configuration for that operation type defined in `MqttReactionConfig`. If the default configuration is also missing, the reaction will publish raw JSON of the query result change.



Query result change processing rules:

1. Determine operation type from the incoming result change event (`ADD`, `Update`, `Delete`, `Aggregation`, `Noop`).
2. Select the matching `MqttCallSpec` from `MqttQueryConfig`.
- if `Aggregation` -> use the `updated` spec.
- if `Noop` -> skip publish (no message generated) and debug log is generated. (Debug)
- For empty `QueryResult` that contains control signals like `bootstrapStarted`, `bootstrapCompleted`, `running` and others, they are ignored with a debug log. (Debug)
3. If the operation-specific spec is missing, fall-back to the default configuration.
4. Generate message body based on the configured `template` template.
5. Publish to `topic` with the configured `qos` and `retain` settings.

#### MQTT v5 and v3.1.1

The reaction targets **MQTT v5** and **MQTT v3.1.1** (fallback option) as main protocols. 

The connection is managed by the [`rumqttc`](https://github.com/bytebeamio/rumqtt) crate, which implements both MQTT v3.1.1 and v5. A single long-lived `AsyncClient` (v5 is tried first) is created at startup and reused for all publish calls. `rumqttc` handles reconnection.

The broker connection is configured through `MqttReactionConfig`:

```rust
#[derive(Clone, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "snake_case")]
pub struct MqttReactionConfig {
    //...... Main config, mqtt-wide config parameters
    /// MQTT broker host address
    #[serde(default = "default_host")]
    pub host: String,

    /// MQTT broker port
    #[serde(default = "default_port")]
    pub port: u16,

    /// Query-specific call configurations
    #[serde(default)]
    pub query_configs: HashMap<String, QueryConfig>,

    /// Default topic template used when no per-query config is provided
    #[serde(default)]
    pub default_template: Option<QueryConfig>,

    /// Identity provider for authentication (takes precedence over user/password)
    #[serde(skip)]
    pub identity_provider: Option<Box<dyn IdentityProvider>>,

    /// Capacity of the async channel
    #[serde(default = "default_event_channel_capacity")] // for client creation and event loop
    pub event_channel_capacity: usize,

    /// Maximum number of retries in row for MQTT operations (e.g., connection, publish, subscribe) before giving up.
    #[serde(
        default = "default_max_retries",
        skip_serializing_if = "Option::is_none"
    )]
    pub max_retries: Option<u32>, // Maximum number of retries in row for MQTT operations upon failure (e.g., connection, publish, subscribe)

    /// Base delay in seconds for retrying MQTT operations after failure. with loop with exponential backoff (delay doubles after each failure up to max_retries)
    #[serde(
        default = "default_base_retry_delay_secs",
        skip_serializing_if = "Option::is_none"
    )]
    pub base_retry_delay_secs: Option<u64>, // Base delay in seconds for retrying MQTT operations upon failure, with exponential backoff

    //...... Shared config parameters (used by both v3 and v5 clients)
    /// MQTT transport protocol (e.g., "tcp", "tls")
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub transport: Option<MqttTransportMode>,

    /// Request (publish, subscribe) channel capacity
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub request_channel_capacity: Option<usize>,

    /// Maximum number of outgoing inflight messages
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub max_inflight: Option<u16>,

    /// Keep alive interval in Seconds (PingReq)
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub keep_alive: Option<u64>,

    /// Clean or Persistent session for MQTT connection (default: true)
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub clean_start: Option<bool>,

    //...... MQTT v3 specific config parameters (if any)
    /// Max incoming packet size
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub max_incoming_packet_size: Option<usize>,

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub max_outgoing_packet_size: Option<usize>,

    //...... MQTT v5 specific config parameters (if any)
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub conn_timeout: Option<u64>, // Connection timeout in milliseconds for MQTT v5

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub connect_properties: Option<MqttConnectProperties>, // MQTT v5 connect properties

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub publish_properties: Option<MqttPublishProperties>, // MQTT v5 publish properties

    //...... Adaptive batching config parameters
    /// Adaptive batching: maximum batch size
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_max_batch_size: Option<usize>,

    /// Adaptive batching: minimum batch size
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_min_batch_size: Option<usize>,

    /// Adaptive batching: maximum wait time in milliseconds
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_max_wait_ms: Option<u64>,

    /// Adaptive batching: minimum wait time in milliseconds
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_min_wait_ms: Option<u64>,

    /// Adaptive batching: throughput window in seconds
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_window_secs: Option<u64>,

    /// Whether adaptive batching is enabled
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub adaptive_enabled: Option<bool>,

    //...... Authentication config parameters
    /// Username for MQTT authentication (if not using identity provider)
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub username: Option<String>,

    /// Password for MQTT authentication (if not using identity provider)
    #[serde(skip_serializing)]
    pub password: Option<String>,
}
```

#### Retrial and Backoff

The reaction implements retrial logic with exponential backoff for for the connection eventloop. The `max_retries` and `base_retry_delay_secs` fields in `MqttReactionConfig` control this behavior.

the delay time is doubled after each failure up to the `max_retries` limit. If the number of consecutive failures exceeds `max_retries`, the reaction gives up and changes its status to `Error`, the defautl value for `max_retries` is 8 retries and the default value for `base_retry_delay_secs` is 1 second.

#### Security

Security is split into two independent axes: **transport security** (how bytes travel) and **identity / authentication** (who is connecting). They can be combined freely.

##### Transport — `MqttTransportMode`

The `transport_mode` field in `MqttReactionConfig` controls the socket layer:

```rust
pub enum MqttTransportMode {
    /// Plain TCP — no encryption (default)
    #[default]
    TCP,
    /// TLS-encrypted transport
    TLS {
        /// CA certificate used to verify the broker's identity
        ca: Vec<u8>,
        /// Optional ALPN protocol list
        alpn: Option<Vec<Vec<u8>>>,
        /// Optional mTLS client certificate and private key.
        client_auth: Option<(Vec<u8>, Vec<u8>)>,
    },
}
```

The `ca` field is required for TLS so the reaction always verifies the broker's certificate. Skipping broker verification is not supported to prevent man-in-the-middle attacks. `client_auth` enables mutual TLS (mTLS), where the reaction presents its own certificate to the broker.

##### Authentication — `IdentityProvider` and Static Credentials

MQTT application-layer authentication (CONNECT username / password) is handled through a pluggable `IdentityProvider` trait rather than a static config field. This allows to deal with credentials from different sources (AWS - Azure - Simple Username-Password) without changing the core publish logic:

```rust
/// Trait for identity providers that supply authentication credentials.
#[async_trait]
pub trait IdentityProvider: Send + Sync {
    /// Fetch credentials for authentication.
    /// Called once during connection setup 
    async fn get_credentials(&self) -> Result<Credentials>;

    /// Clone the provider into a boxed trait object.
    fn clone_box(&self) -> Box<dyn IdentityProvider>;
}
```

`get_credentials` is called on new connection/reconnection attempt.

Also static username/password can be used by implementing using the raw `username` and `password` fields that return fixed credentials, when an `IdentityProvider` is not needed.

#### Integration with Adaptive Batcher

The reaction always processes events through the adaptive batcher, even if adaptive batching is disabled (in which case it behaves like a simple pass-through).

Adaptive batching parameters are added to `MqttReactionConfig` to allow tuning the batch size and timing behavior. The reaction relies on the batcher to group events into batches before processing and publishing.

### API Design

the expected `MqttReaction` structure:

```rust
pub struct MqttReaction {
    base: ReactionBase,
    config: MqttReactionConfig,
    adaptive_config: AdaptiveBatchConfig,
    metrices: MqttReactionMetrices,
    // more fields to be specified during implementation.
}
```

And the expected `MqttReactionBuilder` structure will be following the best practises done by the current reaction builders like Http and Log builders and the Mqtt source builder.

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

## Supportability

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->
Metrics will be exposed, covering:
- Total number of published messages
- Total number of publish failures
- Total number of reconnections.

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
- The Mqtt Source design document: https://github.com/drasi-project/design-documents/blob/main/drasi-lib/Mqtt-Source/mqtt-source-design.md
- The upcoming Shell Reaction design document.

