# Title

* Project Drasi - 2026-3-6 - Ahmed Kamal (@ahmed-kamal2004)

## Overview

MQTT is a widely used lightweight messaging protocol for IoT, industrial telemetry, and event-driven systems. This design adds a new Drasi Source that subscribes to one or more MQTT topics and converts incoming messages into `SourceChange` events that Drasi continuous queries can process in near real time. With this source, teams can connect devices and event publishers that already use MQTT brokers (for example, Mosquitto, EMQX, or HiveMQ) without building custom ingestion bridges.

The MQTT Source will support configurable topic subscriptions, secure broker authentication, payload decoding (JSON first), and topic-to-graph mapping so users can write expressive queries over physical hierarchies such as building/floor/room/sensor. The implementation is expected to follow the existing Drasi source architecture and patterns so that it remains maintainable, testable, and consistent with other sources in drasi-core.


## Terms and definitions

<!-- These terms are internal to your design and will not be shared publicly. -->

| Term | Definition |
|------|------------|
| MQTT | Message Queuing Telemetry Transport, a lightweight publish/subscribe protocol |
| Topic hierarchy | Multi-level MQTT topic path used to organize and route messages (e.g., `building-1/floor-1/room-2/sensor-1`) |
| SourceChange | Internal Drasi change envelope emitted by sources into the continuous query engine |

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->

### User scenarios

**IoT Platform Engineer**

An engineer operates a fleet of sensors publishing telemetry to MQTT topics and wants Drasi to detect patterns in near real time (for example, persistent over-temperature conditions). They need to configure one MQTT Source, subscribe to topic patterns, map topic segments into queryable entities, and run continuous queries without writing custom adapters.

**Data Analyst / Operations User**

An analyst wants to query sensor hierarchy and values (Building → Floor → Room → Sensor) and create reactive logic. They need stable graph labels and properties that reflect MQTT topics and payloads so query authoring is predictable.

### Goals

1. Add an MQTT Source for drasi-core that subscribes to broker topics and emits Drasi `SourceChange` records.
2. Support topic hierarchy mapping to graph elements so users can write meaningful Cypher/GQL queries.
3. Support JSON payloads as the primary format with clear extension points for future formats.
4. Support MQTT v5 as the primary protocol version while remaining compatible with MQTT v3.1.1 brokers.
5. Validate interoperability with target brokers (Mosquitto and HiveMQ).

### Non-Goals

<!-- Describe non-goals to identity something that we won't be focusing on immediately. We won't be expending any effort on these matters. -->

## Design requirements

<!-- Describe key technical requirements, dependencies, and constraints that impact the design.

Provide a link to the issue(s) tracking this work. -->

### Requirements

- Source implementation should align with patterns followed by other drasi-core sources.
- Topic subscription must support exact topics and wildcard forms (`+`, `#`) where broker permits.
- Topic-to-schema mapping must support hierarchical querying use cases.
- Payload decoding must support JSON object payloads and basic scalar-in-object payloads.
- Configuration should allow selecting mapping strategy and payload format.

### Dependencies

- drasi-core source implementation patterns.
- Continuous query engine expectations for node/relationship labels and stable IDs.
- Broker availability and protocol support in test environments (Mosquitto, HiveMQ).

### Out of scope

The first iteration does not include broad non-JSON codec support (for example Protobuf), or advanced broker-specific optimizations. Those can be added in follow-up iterations once production usage patterns are validated.

## Design

### High-level design

The MQTT Source connects to one configured broker, subscribes to configured topic patterns, decodes each incoming payload, then maps topic segments and payload fields into graph-oriented `SourceChange` events. These events are processed by optional middleware (JQ) before being forwarded to Drasi’s query engine.

The key design axis is topic mapping strategy. This document captures three approaches:

1. **Hierarchy model mapping (preferred for readability):** user declares semantic levels such as `Building/Floor/Room/Sensor`.
2. **Separator/regex mapping:** user declares parsing pattern (e.g., dash-separated entity/value tokens).
3. **Level mapping fallback:** no semantic config; levels become generic labels (`L0`, `L1`, `L2`).

### Architecture Diagram

```text
Sensors/Publishers -> MQTT Broker -> MQTT Source -> Mapping/Middleware -> Drasi Query Container -> Reactions
```

### Detail design

#### Input data examples

Example payloads received from publishers:

```json
{
	"timestamp": "2026-03-05T10:15:30Z",
	"type": "temperature",
	"value": 4.2,
	"unit": "C"
}
```

or

```json
{
	"temperature": "4.2"
}
```

Example topic:

```text
building-1/floor-1/room-2/sensor-1
```

#### Topic hierarchy guidance

The source should support structured topic paths that align with industrial hierarchy patterns such as ISA-95 and Unified Namespace style trees. For example:

```text
Enterprise/Site/Area/ProductionLine/WorkCell/Equipment/DataPoint
```

or:

```yaml
manufacturing/
  plantA/
    sensors/
      humidity/
        sensor001
        sensor002
    inventory/
      raw_materials/
        current_stock
      finished_goods/
        current_stock
    quality_control/
      inspection/
        results
```

This allows filtered subscriptions such as:

```text
manufacturing/plantB/quality_control/testing/#
```

#### Query examples

Simple retrieval query:

```cypher
MATCH (b:Building {id: "building-1"})
			-[:HAS_FLOOR]->(f:Floor {id: "floor-2"})
			-[:HAS_SENSOR]->(s:Sensor {id: "sensor-7"})
RETURN b.id AS building, f.id AS floor, s.id AS sensor, s.value
```

Complex condition query:

```cypher
MATCH (b:Building)-[:HAS_FLOOR]->(f:Floor)-[:HAS_ROOM]->(r:Room)-[:HAS_SENSOR]->(s:Sensor)
WHERE s.type = 'temperature'

WITH
	b, f, r, s,
	drasi.changeDateTime(s) AS temperatureChangeTime

WHERE
	temperatureChangeTime != datetime({epochMillis: 0}) AND
	drasi.trueFor(
		s.value > 25,
		duration({ seconds: 15 })
	)

RETURN
	b.id AS buildingId,
	f.id AS floorId,
	r.id AS roomId,
	s.id AS sensorId,
	s.value AS temperature,
	temperatureChangeTime AS fireRiskDetectedSince
```

#### Topic mapping options

##### Option A: User-declared hierarchy model

User config:

```text
Building/Floor/Sensor
```

Topic:

```text
campus/the-first-floor/temperature
```

Mapped labels:

```text
campus -> Building
the-first-floor -> Floor
temperature -> Sensor
```

Pros: highly readable queries and explicit semantics.
Cons: requires mapping model configuration and enforcement.

##### Option B: Separator/regex mapping

User config provides naming pattern (for example `-`).

Topic:

```text
building-1/floor-3/sensor-2
```

Mapped labels:

```text
building-1 -> building
floor-3 -> floor
sensor-2 -> sensor
```

Pros: flexible and compact config.
Cons: naming convention must be strictly followed.

##### Option C: Level mapping (no semantic config)

Topic:

```text
building-1/floor-3/sensor-2
```

Mapped labels:

```text
building-1 -> L0
floor-3 -> L1
sensor-2 -> L2
```

Pros: zero config.
Cons: lower query readability and weaker domain semantics.

#### Behavior

Proposed sensor input shape:
```json
{
	"type": "temperature",
	"value": 4.2,
	"unit": "C"
}
```
With topic name:
```
building-1/floor-1/room-2/sensor-1
```
With configured hierarchical model:
```
Building/Floor/Room/Sensor
```

Proposed Drasi `SourceChange` records emitted after source transformation:

```rust
// Topic segments are mapped positionally onto the configured hierarchy model:
// building-1 -> Building, floor-1 -> Floor, room-2 -> Room, sensor-1 -> Sensor
query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "building-1"),
			labels: Arc::new([Arc::from("Building")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "building-1",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "floor-1"),
			labels: Arc::new([Arc::from("Floor")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "floor-1",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "room-2"),
			labels: Arc::new([Arc::from("Room")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "room-2",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "building-1-has-floor-1"),
			labels: Arc::new([Arc::from("HAS_FLOOR")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "building-1"),
		in_node: ElementReference::new("mqtt", "floor-1"),
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "floor-1-has-room-2"),
			labels: Arc::new([Arc::from("HAS_ROOM")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "floor-1"),
		in_node: ElementReference::new("mqtt", "room-2"),
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "room-2-has-sensor-1"),
			labels: Arc::new([Arc::from("HAS_SENSOR")]),
			effective_from: 0,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "room-2"),
		in_node: ElementReference::new("mqtt", "sensor-1"),
	},
}).await;
```

Main SourceChange:

```rust
query.process_source_change(SourceChange::Update {
    element: Element::Node {
        metadata: ElementMetadata {
            reference: ElementReference::new("mqtt", "sensor-1"),
            labels: Arc::new([Arc::from("Sensor")]),
            effective_from: 0,
        },
        properties: ElementPropertyMap::from(json!({
            "type": "temperature",
            "value": 4.2,
            "unit": "C",
            "topic": "building-1/floor-1/room-2/sensor-1"
        }))
    },
}).await;
```

For the hierarchy model, the source should emit one node per topic segment and one relation per parent-child edge so the topic name becomes a traversable graph structure in Drasi.

If this operation is done on each message, this will decrease the performance.

#### Data format

JSON is the primary payload format in v1. Additional formats may be added later based on explicit configuration.

#### MQTT protocol version

v5 is the preferred implementation target because it offers better session and flow-control features while keeping practical compatibility with v3.1.1 broker ecosystems.

#### Target brokers

- Mosquitto
- HiveMQ

<!-- This section should be detailed and through enough that another developer could implement your design and provide enough detail to get a high confidence estimate of the cost to implement the feature but isn't as detailed as the code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and describe it in enough detail that someone else could implement it. Cover ALL of the important decisions like names. Your goal is to get agreement to proceed with coding and PRs.

If there are alternative you are considering please include that in the open questions section.

If the product has a layered architecture, it's good to align these sections with the product's layers. This will help readers use their current understanding to understand your ideas.

- **Advantages of this design** - Describe what's good about this plan relative to other options. Does it feel easy to implement? Provide flexibility for future work?
- **Disadvantages** - Describe what's not ideal about this plan. If you don't point these things out other people will do it for you. This is a good place to cover risks. -->

### API Design

No new REST API is proposed in this design. The expected change is a new source type configuration surface (broker endpoint/auth, subscriptions, mapping strategy, payload format) through existing Drasi source resource flows and CLI/apply workflows.

### Alternatives Considered

- **Hierarchy model mapping** vs **separator/regex mapping** vs **generic level mapping**.
- Hierarchy model is favored for readability of user queries.
- Separator/regex and level mapping remain valid fallback modes for low-governance topic ecosystems.

## Security

- Use secure broker connectivity (TLS) and support credential-based auth mechanisms.
- Validate and safely parse payloads to prevent malformed-input failures.

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc. -->

## Compatibility impact

- Compatible with brokers that support MQTT v5 and v3.1.1.
- No breaking changes expected for existing sources; this is an additive source type.
- Query compatibility depends on selected mapping strategy (semantic labels vs generic level labels).

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

## Supportability

### Telemetry
- TODO

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->

### Verification

- Unit tests for drasi source.
- Integration tests against Mosquitto and HiveMQ (needs more research).
- End-to-end test scenarios validating query outcomes for hierarchy traversal and temporal conditions (needs more research).

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

## Development Plan

1. TODO

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

## Open issues

1. Should hierarchy model mapping be mandatory in v1, or optional with auto-fallback?
2. Which additional payload formats should be prioritized after JSON?

<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design. -->

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

## References

- https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/
- https://drasi.io/drasi-kubernetes/tutorials/curbside-pickup/
- https://www.machbase.com/en/post/how-to-make-sensor-data-send-directly-to-a-database-via-mqtt
- https://drasi.io/reference/query-language/
- https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901118
- https://www.hivemq.com/blog/mqtt5-essentials-part6-user-properties/
- https://www.isa.org/standards-and-publications/isa-standards/isa-95-standard

<!-- Optional. Add the design documents and references you use for this document. -->

