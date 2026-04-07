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
2. Configurable MQTTCallSpec (topic - QoS - retain) based on per query and operation.
3. Support MQTT v3.1.1 as the primary protocol version, this achieves (maximum compatibility) as it understands MQTT v3.1.1 brokers and interoperable with MQTT v5 brokers.
4. Validate interoperability with target brokers (Mosquitto and HiveMQ).

### Non-Goals

## Design requirements

### Requirements

- Reaction implementation should align with patterns followed by other drasi-core reactions.
- Payload decoding must support JSON object payloads, basic scalar-in-object payloads or any other body format.
- Reaction implementation must support mqtt-reaction-wide configuration via reaction spec parameters.
- Reaction implementation must support QoS and retain configuration per query and operation.
- Must avoid breaking existing reaction contract semantics.
- Must support TLS/SSL transport and identity-based authentication.

### Dependencies

- drasi-core reaction implementation patterns.
- Broker availability and protocol support in test environments (Mosquitto, HiveMQ).

### Out of scope

- Retry mechanism for unknown failures.
- This first iteration doesn't include `DLQ` (Dead-letter topic used to store failed publish events after retry exhaustion ).

## Design

### High-level design

The MQTT Reaction receives query result events from the Drasi runtime and transforms each event into a publish request composed of:
- target MQTT topic
- payload
- MQTT publish options (QoS, retain)

The reaction maintains a broker connection with reconnect logic. Events are  processed then published to the broker.

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

#### 1. Event-to-Topic Mapping and Payload template

Introduce new struct `MqttCallSpec` that includes the target topic name `topic`, the message template body `body`, retaining method `retain` and the Quality of Service `qos`.

```rust
pub struct MqttCallSpec {
    /// MQTT topic.
    pub topic: String,

    /// Request body as a Handlebars template.
    /// If empty, sends the raw JSON data.
    #[serde(default)]
    pub body: String,

    /// MQTT message retain policy.
    #[serde(default)]
    pub retain: RetainPolicy,

    /// QoS level for MQTT messages.
    #[serde(default)]
    pub qos: QualityOfService,
}
```

Example for using `MqttQueryConfig` for 

```rust
let default_template = MqttQueryConfig {
        added: Some(MqttCallSpec {
            topic: "t/added".to_string(),
            body: "[{{query_name}}] + {{after.symbol}}: ${{after.price}}".to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
        updated: Some(MqttCallSpec {
            topic: "t/updated".to_string(),
            body: "[{{query_name}}] ~ {{after.symbol}}: ${{before.price}} -> ${{after.price}}"
                .to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
        deleted: Some(MqttCallSpec {
            topic: "t/deleted".to_string(),
            body: "[{{query_name}}] - {{before.symbol}} removed".to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
    };
```

#### 3. Connection and Authentication

Supported configuration: