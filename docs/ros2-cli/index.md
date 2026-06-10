# ROS 2 CLI

Agnocast ships a `ros2agnocast` package that extends the standard `ros2` CLI with Agnocast-aware verbs. This page describes:

- which standard `ros2` commands work with Agnocast endpoints, and
- how to use the Agnocast-specific commands (`ros2 topic list_agnocast`, `ros2 node info_agnocast`, …).

## Support Status of ros2 Commands for Agnocast

This section enumerates the standard `ros2` CLI commands shipped with `ros2cli` (and related packages such as `ros2bag`, `ros2lifecycle`) and describes how they behave in an Agnocast environment.

Columns used in the tables below:

- **Works as-is** — whether the unmodified `ros2` command produces useful results for Agnocast endpoints.
  - ✓ Works without modification (either the command does not depend on Agnocast semantics, or the information it needs is available on DDS).
  - ⚠ Works partially — typically, DDS-visible aspects (bridged topics, `rclcpp::Node`-based nodes that use Agnocast pub/sub, etc.) are reported correctly, but purely Agnocast-only endpoints are invisible.
  - ✗ Does not see Agnocast-only endpoints because Agnocast bypasses the RMW/DDS layer.
  - N/A Not applicable (the command is Agnocast-specific and has no upstream `ros2` counterpart).
- **Agnocast version** — whether `ros2agnocast` provides a dedicated Agnocast-aware verb (e.g. `ros2 topic list_agnocast`, `ros2 agnocast generate-bridge-plugins`). ✓ means a variant exists; ✗ means no variant exists today. The verb-level tables below mark each verb's **scope**:
  - **Cluster-wide** — shows Agnocast endpoints across all IPC namespaces and ECUs on the same `ROS_DOMAIN_ID` wherever an Agnocast discovery agent runs. Without a discovery agent, some commands do not work at all; others fall back to showing only the current IPC namespace.
  - **Local IPC namespace** — inspects only the IPC namespace the command runs in.
- **Planned** — whether dedicated Agnocast support is intended.
  - `Yes` Agnocast-specific support is planned.
  - `No` Explicitly not planned (the command is DDS-only and has no Agnocast counterpart).
  - `TBD` Not decided.
  - `-` Not applicable — the command already works as-is (including via the bridge), so no dedicated Agnocast support is needed.
- **Notes** — short explanation and pointers.

### 1. Top-level ros2 commands

| Command | Works as-is | Agnocast version | Planned | Notes |
|---------|:-----------:|:----------------:|:-------:|-------|
| `ros2 action` | ✗ | ✗ | TBD | Agnocast does not implement Actions (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md)). Pure Agnocast nodes are not visible. |
| `ros2 bag` | ⚠ | ✓ (`record_agnocast`) | - | Standard `ros2 bag record` works only when a bridge is already active; Agnocast-only topics are not captured without one. The `_agnocast` variant extends `record` with full Agnocast awareness. See §9. |
| `ros2 component` | ✓ | ✗ | - | Component Container loading works for `agnocast::Node`. See "Composable Node Considerations" in [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md). |
| `ros2 daemon` | ✓ | ✗ | - | DDS discovery daemon; unrelated to Agnocast's shared memory path. |
| `ros2 doctor` / `ros2 wtf` | ✓ | ✗ | TBD | Reports DDS/RMW health only. No Agnocast-specific diagnostics yet. |
| `ros2 interface` | ✓ | ✗ | - | Inspects message/service/action **definitions** (`show`, `list`, `proto`, `package`, `packages`) via the ament index. It does not touch any running node or transport, so Agnocast-defined types in user packages are listed and shown like any other ROS 2 type. |
| `ros2 launch` | ✓ | ✗ | - | Launches processes; Agnocast executables are launched the same way. |
| `ros2 lifecycle` | ✗ | ✗ | TBD | `agnocast::Node` does not support lifecycle (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md) §4.3). |
| `ros2 multicast` | ✓ | ✗ | - | Exercises DDS multicast; unrelated to Agnocast. |
| `ros2 node` | ⚠ | ✓ (`list_agnocast`, `info_agnocast`) | - | `as-is` does not see pure `agnocast::Node` instances at all — only `rclcpp::Node`-based nodes (including those that use Agnocast pub/sub) appear. Use the `_agnocast` verbs to include `agnocast::Node`. See §2. |
| `ros2 param` | ✓ | ✗ | - | `agnocast::Node` now instantiates a `ParameterService` during construction, so the standard parameter verbs work against Agnocast nodes the same as against `rclcpp::Node`. |
| `ros2 pkg` | ✓ | ✗ | - | Package metadata operations (`create`, `list`, `prefix`, `executables`, `xml`) read from the colcon/ament install layout and `package.xml`. Transport-agnostic and works the same for Agnocast packages. |
| `ros2 run` | ✓ | ✗ | - | Launches an executable; no Agnocast-specific behavior. |
| `ros2 security` | N/A | ✗ | No | Manages SROS2 / DDS-Security artifacts (enclaves, keystores, permission/policy files) that authenticate and encrypt DDS traffic. **Has no effect on Agnocast pub/sub**: communication between Agnocast endpoints goes through shared memory, not DDS, so SROS2 policies neither authorize nor restrict it. The commands still run, but generated artifacts only protect the DDS side (including the Agnocast↔ROS 2 bridge). |
| `ros2 service` | ✗ | ✗ | TBD | Agnocast services use internal shared-memory topics prefixed with `/AGNOCAST_SRV_*` and are not exposed via DDS. Agnocast services are marked experimental (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md) §2.6). |
| `ros2 topic` | ⚠ | ✓ (`list_agnocast`, `info_agnocast`, `echo_agnocast`, `hz_agnocast`, `delay_agnocast`) | - | Standard `ros2 topic` verbs have limited visibility into Agnocast: pure Agnocast-only topics are invisible, and bridged topics are seen only from the DDS side. The `_agnocast` variants extend each verb with full Agnocast awareness. See §3. |
| `ros2 agnocast` | N/A | ✓ (new top-level command) | - | Provided by `ros2agnocast` for Agnocast-specific operations (`version`, `generate-bridge-plugins`, `discovery-daemon-status`, `bridge-daemon-status`). See §10. |

### 2. `ros2 node` verbs

| Verb | Works as-is | Agnocast version | Scope of Agnocast verb | Planned | Notes |
|------|:-----------:|:----------------:|:----------------------:|:-------:|-------|
| `ros2 node list` | ⚠ | ✓ `list_agnocast` | Cluster-wide | - | `as-is` lists only `rclcpp::Node`-based nodes, not pure `agnocast::Node` instances. `list_agnocast` also lists `agnocast::Node` instances, wherever a discovery agent runs (across IPC namespaces and ECUs); otherwise only the current IPC namespace. Supports `-a`, `-c`, and `-d` (`-d` includes internal `agnocast_bridge_node_*`). |
| `ros2 node info` | ⚠ | ✓ `info_agnocast` | Cluster-wide | - | `as-is` cannot target a pure `agnocast::Node` (it is absent from `ros2 node list`), and does not label Agnocast publishers/subscribers on the nodes it can show. `info_agnocast` adds those Agnocast publishers/subscribers and bridge-status labels, wherever a discovery agent runs (across IPC namespaces and ECUs); otherwise only the current IPC namespace. |

### 3. `ros2 topic` verbs

| Verb | Works as-is | Agnocast version | Scope of Agnocast verb | Planned | Notes |
|------|:-----------:|:----------------:|:----------------------:|:-------:|-------|
| `ros2 topic list` | ⚠ | ✓ `list_agnocast` | Cluster-wide | - | `as-is` lists only topics with DDS endpoints, so pure Agnocast-only topics are missing and bridged topics carry no Agnocast marking. `list_agnocast` adds Agnocast-only topics and tags Agnocast topics `(Agnocast enabled)` / `(Agnocast enabled, bridged)`, wherever a discovery agent runs (across IPC namespaces and ECUs); otherwise only the current IPC namespace. See [Topic List](#topic-list) below. |
| `ros2 topic info` | ⚠ | ✓ `info_agnocast` | Cluster-wide | - | `as-is` cannot target a pure Agnocast-only topic (it is absent from `ros2 topic list`), and shows only DDS publishers/subscribers for bridged topics. `info_agnocast` adds Agnocast publisher/subscriber counts, QoS and message type, wherever a discovery agent runs (across IPC namespaces and ECUs); otherwise only the current IPC namespace. Supports `-v` and `-d`. See [Topic Info](#topic-info) below. |
| `ros2 topic echo` | ⚠ | ✓ `echo_agnocast` | Cluster-wide | - | `as-is` cannot target a pure Agnocast-only topic and works only when a bridge is already active; the message type does not need to be specified explicitly. `echo_agnocast` forces an A2R bridge and then delegates to `ros2 topic echo` — all `ros2 topic echo` arguments (e.g. `--filter`, `--once`, `--no-arr`) are supported. Also works for standard ROS 2 topics. Without a discovery agent, Agnocast-only topics cannot be observed. See [Topic Echo](#topic-echo) below. |
| `ros2 topic pub` | ⚠ | ✗ | — | TBD | Publishes via DDS, so messages reach Agnocast subscribers only via the bridge. The `--wait-matching-subscriptions` option does **not** work correctly for Agnocast subscribers — DDS discovery counts only the bridge-side DDS subscription (if any), not the Agnocast subscribers behind it. |
| `ros2 topic hz` | ⚠ | ✓ `hz_agnocast` | Cluster-wide | - | `as-is` cannot target a pure Agnocast-only topic and works only when a bridge is already active. `hz_agnocast` forces an A2R bridge and then delegates to `ros2 topic hz` — all `ros2 topic hz` arguments (e.g. `--filter`) are supported. Also works for standard ROS 2 topics. Without a discovery agent, Agnocast-only topics cannot be observed. See [Topic Hz](#topic-hz) below. |
| `ros2 topic bw` | ✗ | ✗ | — | TBD | Requires subscribing to the topic. |
| `ros2 topic delay` | ⚠ | ✓ `delay_agnocast` | Cluster-wide | - | `as-is` cannot target a pure Agnocast-only topic and works only when a bridge is already active. `delay_agnocast` forces an A2R bridge and then delegates to `ros2 topic delay` — all `ros2 topic delay` arguments (e.g. `-w`) are supported. Also works for standard ROS 2 topics. Without a discovery agent, Agnocast-only topics cannot be observed. See [Topic Delay](#topic-delay) below. |
| `ros2 topic type` | ⚠ | ✗ | — | No | Resolves a topic's message type via the DDS graph, which works for bridged topics (their DDS endpoints carry the type). No Agnocast `type` verb is planned, but `ros2 topic info_agnocast <topic>` now reports the type of a pure Agnocast-only topic when a discovery agent is running, showing `<UNKNOWN>` only if none has reported one. |
| `ros2 topic find` | ⚠ | ✗ | — | No | Lists topics of a given type via the DDS graph. Same constraint as `type`: bridged topics are findable; pure Agnocast-only topics are not, because no type→name index exists on the Agnocast side. |

### 4. `ros2 service` verbs

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 service list` | ✗ | ✗ | TBD | |
| `ros2 service call` | ✗ | ✗ | TBD | |
| `ros2 service type` / `find` / `info` / `echo` | ✗ | ✗ | TBD | |

### 5. `ros2 param` verbs

`agnocast::Node` instantiates a `ParameterService` during construction, so the standard `ros2 param` verbs operate on `agnocast::Node` exactly as on `rclcpp::Node`. No Agnocast-specific verb is needed.

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 param list` | ✓ | ✗ | - | Works for both `rclcpp::Node` and `agnocast::Node`. |
| `ros2 param get` | ✓ | ✗ | - | Same as above. |
| `ros2 param set` | ✓ | ✗ | - | Same as above. |
| `ros2 param describe` | ✓ | ✗ | - | Same as above. |
| `ros2 param dump` | ✓ | ✗ | - | Same as above. |
| `ros2 param load` | ✓ | ✗ | - | Same as above. |
| `ros2 param delete` | ✓ | ✗ | - | Same as above. |

### 6. `ros2 action` verbs

`agnocast::Node` does not provide an Actions implementation. Pure Agnocast nodes are not discovered by DDS action discovery.

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 action list` | ✗ | ✗ | TBD | Agnocast has no Action implementation. |
| `ros2 action info` | ✗ | ✗ | TBD | Same as above. |
| `ros2 action send_goal` | ✗ | ✗ | TBD | Same as above. |
| `ros2 action echo` | ✗ | ✗ | TBD | Same as above. |

### 7. `ros2 lifecycle` verbs

`agnocast::Node` does not support Lifecycle (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md) §4.3).

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 lifecycle nodes` | ✗ | ✗ | TBD | No lifecycle support in Agnocast. |
| `ros2 lifecycle list` | ✗ | ✗ | TBD | Same as above. |
| `ros2 lifecycle get` | ✗ | ✗ | TBD | Same as above. |
| `ros2 lifecycle set` | ✗ | ✗ | TBD | Same as above. |

### 8. `ros2 component` verbs

Component containers can load `agnocast::Node` subclasses; the container itself is a DDS participant (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md) §5).

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 component list` | ✓ | ✗ | - | Operates on DDS-visible containers. |
| `ros2 component load` | ✓ | ✗ | - | Loading an Agnocast-enabled component into a container is supported. |
| `ros2 component unload` | ✓ | ✗ | - | |
| `ros2 component types` | ✓ | ✗ | - | Lists registered component plugin types via `ament_index` (`rclcpp_components` resources); transport-agnostic. |
| `ros2 component standalone` | ✓ | ✗ | - | Launches a component in a self-contained container process. Works for `agnocast::Node`, with the same caveat as Composable Node loading — the container becomes a DDS participant (see [agnocast_node_interface_comparison.md](https://github.com/autowarefoundation/agnocast/blob/main/docs/agnocast_node_interface_comparison.md) §5). |

### 9. `ros2 bag` verbs

`ros2 bag record` captures DDS traffic only; Agnocast-only topics live in shared memory and are not captured unless the Agnocast↔ROS 2 bridge is active. `record_agnocast` watches for Agnocast topics and automatically creates the required A2R bridge, making them visible to DDS and therefore recordable.

| Verb | Works as-is | Agnocast version | Scope of Agnocast verb | Planned | Notes |
|------|:-----------:|:----------------:|:----------------------:|:-------:|-------|
| `ros2 bag record` | ⚠ | ✓ `record_agnocast` | Cluster-wide | - | `as-is` cannot target a pure Agnocast-only topic and works only when a bridge is already active. `record_agnocast` runs a component that continuously watches for Agnocast topics and automatically creates the corresponding A2R bridge when they appear, while concurrently running `ros2 bag record` — all `ros2 bag record` arguments (e.g. `-e`) are supported. Also works for standard ROS 2 topics. Without a discovery agent, Agnocast-only topics cannot be recorded. See [Bag Record](#bag-record) below. |
| `ros2 bag play` | ✓ | ✗ | — | - | Works as-is. Messages originally published by Agnocast publishers are replayed as ROS 2 (DDS) publishers. If Agnocast subscribers exist for the topic, an R2A bridge is created automatically, so playback reaches them without any extra steps. |
| `ros2 bag info` | ✓ | ✗ | — | - | Operates on a stored bag file, independent of transport. |
| `ros2 bag list` | ✓ | ✗ | — | - | Same as above. |
| `ros2 bag convert` | ✓ | ✗ | — | - | Same as above. |
| `ros2 bag reindex` | ✓ | ✗ | — | - | Same as above. |
| `ros2 bag burst` | ✓ | ✗ | — | - | Same as `play`. |

### 10. `ros2 agnocast` verbs

`ros2agnocast` registers a top-level `ros2 agnocast` command with the following verbs (see [`src/ros2agnocast/setup.py`](https://github.com/autowarefoundation/agnocast/blob/main/src/ros2agnocast/setup.py)).

| Verb | Scope | Purpose |
|------|:-----:|---------|
| `ros2 agnocast --version` / `-v` | Local host only | Print version information for Agnocast components installed on the host where the command is run (userland libraries and the loaded kernel module). |
| `ros2 agnocast version` | Local host only | Same as `--version` (verb form). |
| `ros2 agnocast generate-bridge-plugins` | Build-time | Generate a ROS 2 bridge plugin package for user message types. Operates on local source/install trees, not on any running system. |
| `ros2 agnocast discovery-daemon-status` | Local IPC namespace | Check that the Agnocast discovery agent for the **current** IPC namespace is running and healthy. Exits non-zero if it is not. Add `-v` for per-check detail. |
| `ros2 agnocast bridge-daemon-status` | Local IPC namespace | Check that the Agnocast bridge daemon process for the **current** IPC namespace is running and healthy. Exits non-zero if it is not. Add `-v` for per-check detail. |

---

## How to use ros2 command for Agnocast

The following sections show usage examples for the Agnocast-specific verbs:

- `ros2 topic list_agnocast`
- `ros2 topic info_agnocast /topic_name`
- `ros2 topic echo_agnocast /topic_name`
- `ros2 topic hz_agnocast /topic_name`
- `ros2 topic delay_agnocast /topic_name`
- `ros2 node list_agnocast`
- `ros2 node info_agnocast /node_name`
- `ros2 bag record_agnocast`

!!! info "Scope"
    `list_agnocast` / `info_agnocast` show Agnocast endpoints wherever an Agnocast discovery agent runs, across IPC namespaces and ECUs on the same `ROS_DOMAIN_ID`; otherwise they show only the current IPC namespace. The DDS side is cluster-wide as usual.

### Topic List

To list all topics including Agnocast, use `ros2 topic list_agnocast`.

The `(Agnocast enabled)` marking covers every IPC namespace and ECU where a discovery agent runs; otherwise it reflects only the current IPC namespace.

If a topic is Agnocast enabled, it will be shown with a "(Agnocast enabled)" suffix.

```bash
$ ros2 topic list_agnocast
/topic_name1
/topic_name2 (Agnocast enabled)
/topic_name3
```

If you want to get only Agnocast related topics, use `ros2 topic list_agnocast | grep Agnocast`:

```bash
$ ros2 topic list_agnocast | grep Agnocast
/topic_name2 (Agnocast enabled)
```

If an Agnocast topic is bridged to ROS 2 publishers or subscribers, it is shown with the "(Agnocast enabled, bridged)" suffix.

```bash
$ ros2 topic list_agnocast
/topic_name1 (Agnocast enabled)
/topic_name2 (Agnocast enabled, bridged)
/topic_name3
```

The `(Agnocast enabled, bridged)` suffix means **the topic has both Agnocast endpoints and ROS 2 endpoints, and a bridge process is forwarding between them**. All three conditions are required:

- at least one Agnocast publisher or subscriber on the topic,
- at least one ROS 2 (DDS-side) endpoint on the other direction (e.g. an Agnocast publisher with a ROS 2 subscriber, or vice versa), and
- a bridge process (standard or performance) currently running for that topic.

If any one of these is missing, the suffix is just `(Agnocast enabled)`. For example, a topic with only an Agnocast publisher and no ROS 2 subscriber is shown as `(Agnocast enabled)` regardless of whether a bridge process is running — there is simply no DDS-side counterpart to bridge to. Table 1 below enumerates every combination.

#### Table 1: Pub/Sub situations and display names

| pub | sub | bridge | display |
| :--- | :--- | :--- | :--- |
| `rclcpp::publisher` | `rclcpp::subscription` | off / standard / performance | `/my_topic` |
| `agnocast::publisher` | `rclcpp::subscription` | off | `/my_topic (WARN: one or more necessary bridges are not running)` |
| `agnocast::publisher` | `rclcpp::subscription` | standard / performance | `/my_topic (Agnocast enabled, bridged)` |
| `rclcpp::publisher` | `agnocast::subscription` | off | `/my_topic (WARN: one or more necessary bridges are not running)` |
| `rclcpp::publisher` | `agnocast::subscription` | standard / performance | `/my_topic (Agnocast enabled, bridged)` |
| `agnocast::publisher` | `agnocast::subscription` | off / standard / performance | `/my_topic (Agnocast enabled)` |
| `agnocast::publisher` | none | off / standard / performance | `/my_topic (Agnocast enabled)` |
| none | `agnocast::subscription` | off / standard / performance | `/my_topic (Agnocast enabled)` |

#### Notes

- If an Agnocast topic and a ROS 2 topic share the same name but have different message types, the status will still be displayed as (Agnocast enabled, bridged). However, in this case, the actual communication will not be established.

### Topic Info

To show the topic info including Agnocast, use `ros2 topic info_agnocast /topic_name`.

The Agnocast publisher/subscriber details below cover every IPC namespace and ECU where a discovery agent runs; otherwise they reflect only the current IPC namespace.

```bash
$ ros2 topic info_agnocast /my_topic
Type: agnocast_sample_interfaces/msg/DynamicSizeArray
ROS 2 Publisher count: 0
Agnocast Publisher count: 1
ROS 2 Subscription count: 0
Agnocast Subscription count: 1
```

For more details, use `--verbose` or `-v`.

```bash
$ ros2 topic info_agnocast /my_topic -v
Type: agnocast_sample_interfaces/msg/DynamicSizeArray

ROS 2 Publisher count: 0
Agnocast Publisher count: 1

Node name: talker_node
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: PUBLISHER (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE

ROS 2 Subscription count: 0
Agnocast Subscription count: 1

Node name: listener_node
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: SUBSCRIPTION (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE
```

#### Debug Mode

By default, `ros2 topic info_agnocast` hides internal bridge nodes and endpoints to provide a cleaner view. To include these internal details, use the `--debug` or `-d` flag.

```bash
$ ros2 topic info_agnocast /my_topic -d
Type: agnocast_sample_interfaces/msg/DynamicSizeArray
ROS 2 Publisher count: 1
Agnocast Publisher count: 2
ROS 2 Subscription count: 1
Agnocast Subscription count: 2
```

You can combine `-d` with `-v` for full verbose output including bridge node details:

```bash
$ ros2 topic info_agnocast /my_topic -v -d
Type: agnocast_sample_interfaces/msg/DynamicSizeArray

ROS 2 Publisher count: 1
Agnocast Publisher count: 2

Node name: agnocast_bridge_node_86050
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: PUBLISHER
GID: 01.10.b2.d2.ef.55.13.0d.74.cc.0c.e6.00.00.16.03.00.00.00.00.00.00.00.00
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (10)
  Durability: TRANSIENT_LOCAL
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite

Node name: agnocast_bridge_node_86050
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: PUBLISHER (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (10)
  Durability: TRANSIENT_LOCAL

Node name: talker_node
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: PUBLISHER (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE

ROS 2 Subscription count: 1
Agnocast Subscription count: 2

Node name: agnocast_bridge_node_86050
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: SUBSCRIPTION
GID: 01.10.b2.d2.ef.55.13.0d.74.cc.0c.e6.00.00.15.04.00.00.00.00.00.00.00.00
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite

Node name: listener_node
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: SUBSCRIPTION (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE

Node name: agnocast_bridge_node_86050
Node namespace: /
Topic type: agnocast_sample_interfaces/msg/DynamicSizeArray
Endpoint type: SUBSCRIPTION (Agnocast enabled)
QoS profile:
  History (Depth): KEEP_LAST (1)
  Durability: VOLATILE
```

### Topic Echo

`echo_agnocast` is an Agnocast extension of `ros2 topic echo`. It forces an Agnocast-to-ROS 2 (A2R) bridge for the specified topic and then delegates to `ros2 topic echo`. All standard `ros2 topic echo` arguments (e.g. `--filter`, `--once`, `--no-arr`) are passed through unchanged. It also works for standard ROS 2 topics.

`echo_agnocast` works cluster-wide wherever a discovery agent runs (across IPC namespaces and ECUs on the same `ROS_DOMAIN_ID`); without a discovery agent, Agnocast-only topics cannot be observed.

When the discovery agent is running:

```bash
$ ros2 topic echo_agnocast /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
[*] Starting ros2cli command...
id: 870
data:
- 870
- 871
- 872
- 873
[...]
```

Without a discovery agent, the command exits with an error:

```bash
$ ros2 topic echo_agnocast /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
NOTE: no /_agnocast_discovery agent visible; showing local NS only via ioctl. Start one with `ros2 run ros2agnocast_discovery_agent discovery_agent` to see other NSes / ECUs.
ERROR: Could not resolve message type for '/my_topic'. The topic was not found in the ROS 2 graph or via /_agnocast_discovery. Make sure the topic exists and, for Agnocast topics, the discovery agent is running.
```

Standard `ros2 topic echo` CLI options are passed through:

```bash
$ ros2 topic echo_agnocast --once --no-arr --filter "m.id % 50 == 0" /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
[*] Starting ros2cli command...
id: 4450
data: '<sequence type: int64, length: 128000>'
---
```

### Topic Hz

`hz_agnocast` is an Agnocast extension of `ros2 topic hz`. It forces an Agnocast-to-ROS 2 (A2R) bridge for the specified topic and then delegates to `ros2 topic hz`. All standard `ros2 topic hz` arguments (e.g. `--filter`) are passed through unchanged. It also works for standard ROS 2 topics.

`hz_agnocast` works cluster-wide wherever a discovery agent runs (across IPC namespaces and ECUs on the same `ROS_DOMAIN_ID`); without a discovery agent, Agnocast-only topics cannot be observed.

When the discovery agent is running:

```bash
$ ros2 topic hz_agnocast /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
[*] Starting ros2cli command...
average rate: 9.832
        min: 0.101s max: 0.103s std dev: 0.00049s window: 11
average rate: 10.037
        min: 0.052s max: 0.103s std dev: 0.01052s window: 22
average rate: 9.977
        min: 0.052s max: 0.103s std dev: 0.00878s window: 32
[...]
```

Without a discovery agent, the command exits with an error:

```bash
$ ros2 topic hz_agnocast /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
NOTE: no /_agnocast_discovery agent visible; showing local NS only via ioctl. Start one with `ros2 run ros2agnocast_discovery_agent discovery_agent` to see other NSes / ECUs.
ERROR: Could not resolve message type for '/my_topic'. The topic was not found in the ROS 2 graph or via /_agnocast_discovery. Make sure the topic exists and, for Agnocast topics, the discovery agent is running.
```

Standard `ros2 topic hz` CLI options are passed through:

```bash
$ ros2 topic hz_agnocast --filter "m.id % 10 == 0" /my_topic
[*] Triggering A2R bridge for '/my_topic'. This may take a few seconds...
[*] Starting ros2cli command...
average rate: 0.990
        min: 1.009s max: 1.011s std dev: 0.00077s window: 2
average rate: 1.002
        min: 0.960s max: 1.011s std dev: 0.02200s window: 4
average rate: 1.000
        min: 0.960s max: 1.011s std dev: 0.02022s window: 5
[...]
```

### Topic Delay

`delay_agnocast` is an Agnocast extension of `ros2 topic delay`. It forces an Agnocast-to-ROS 2 (A2R) bridge for the specified topic and then delegates to `ros2 topic delay`. All standard `ros2 topic delay` arguments (e.g. `-w`) are passed through unchanged. It also works for standard ROS 2 topics.

`delay_agnocast` works cluster-wide wherever a discovery agent runs (across IPC namespaces and ECUs on the same `ROS_DOMAIN_ID`); without a discovery agent, Agnocast-only topics cannot be observed.

When the discovery agent is running:

```bash
$ ros2 topic delay_agnocast /pose_chatter
[*] Triggering A2R bridge for '/pose_chatter'. This may take a few seconds...
[*] Starting ros2cli command...
average delay: 0.001
        min: 0.001s max: 0.001s std dev: 0.00000s window: 1
average delay: 0.001
        min: 0.001s max: 0.001s std dev: 0.00014s window: 3
average delay: 0.001
        min: 0.001s max: 0.001s std dev: 0.00012s window: 5
[...]
```

Without a discovery agent, the command exits with an error:

```bash
$ ros2 topic delay_agnocast /pose_chatter
[*] Triggering A2R bridge for '/pose_chatter'. This may take a few seconds...
NOTE: no /_agnocast_discovery agent visible; showing local NS only via ioctl. Start one with `ros2 run ros2agnocast_discovery_agent discovery_agent` to see other NSes / ECUs.
ERROR: Could not resolve message type for '/pose_chatter'. The topic was not found in the ROS 2 graph or via /_agnocast_discovery. Make sure the topic exists and, for Agnocast topics, the discovery agent is running.
```

Standard `ros2 topic delay` CLI options are passed through:

```bash
$ ros2 topic delay_agnocast -w 1 /pose_chatter
[*] Triggering A2R bridge for '/pose_chatter'. This may take a few seconds...
[*] Starting ros2cli command...
average delay: 0.002
        min: 0.002s max: 0.002s std dev: 0.00000s window: 1
average delay: 0.001
        min: 0.001s max: 0.001s std dev: 0.00000s window: 1
average delay: 0.001
        min: 0.001s max: 0.001s std dev: 0.00000s window: 1
[...]
```

### Node List

To list all nodes including those implemented with Agnocast, use `ros2 node list_agnocast`.

`agnocast::Node` instances are listed for every IPC namespace and ECU where a discovery agent runs; otherwise only for the current IPC namespace.

Standard ROS 2 nodes are displayed normally, while `agnocast::Node` instances are marked with the "(Agnocast enabled)" suffix.

```bash
$ ros2 node list_agnocast
/ros2_talker_node
/agnocast_listener_node (Agnocast enabled)
```

#### Debug Mode

By default, `ros2 node list_agnocast` hides internal bridge nodes and endpoints to provide a cleaner view. To include these internal details, use the `--debug` or `-d` flag.

```bash
$ ros2 node list_agnocast -d
/ros2_talker_node
/agnocast_bridge_node_86050
/agnocast_listener_node (Agnocast enabled)
```

#### Notes

Detection of `agnocast::Node` instances depends on the presence of Agnocast-enabled endpoints. A node without at least one Agnocast publisher or subscriber will be omitted from the output.

### Node Info

To show the node info including Agnocast, use `ros2 node info_agnocast /node_name`.

Agnocast publisher/subscriber details are reported for a node in any IPC namespace or ECU where a discovery agent runs; otherwise only for the current IPC namespace.

```bash
$ ros2 node info_agnocast /listener_node
  Subscribers:
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /my_topic: agnocast_sample_interfaces/msg/DynamicSizeArray (Agnocast enabled, bridged)
  Publishers:
    /my_topic2: agnocast_sample_interfaces/msg/DynamicSizeArray (Agnocast enabled)
    /parameter_events: rcl_interfaces/msg/ParameterEvent
    /rosout: rcl_interfaces/msg/Log
  Service Servers:
    /listener_node/describe_parameters: rcl_interfaces/srv/DescribeParameters
    /listener_node/get_parameter_types: rcl_interfaces/srv/GetParameterTypes
    /listener_node/get_parameters: rcl_interfaces/srv/GetParameters
    /listener_node/list_parameters: rcl_interfaces/srv/ListParameters
    /listener_node/set_parameters: rcl_interfaces/srv/SetParameters
    /listener_node/set_parameters_atomically: rcl_interfaces/srv/SetParametersAtomically
  Service Clients:
  Action Servers:
  Action Clients:
```

Similar to `ros2 topic list_agnocast`, the (Agnocast enabled, bridged) suffix in the node info indicates that communication has been successfully established between Agnocast and ROS 2 for that specific topic.

If a topic is Agnocast-enabled but not currently bridged (e.g., there is no corresponding ROS 2 publisher/subscriber or the bridge process is not active), it will be displayed simply as (Agnocast enabled). This allows you to verify the connectivity status of each topic directly from the node's perspective.

### Bag Record

`record_agnocast` is an Agnocast extension of `ros2 bag record`. It runs a component that continuously watches for Agnocast topics and automatically creates the corresponding A2R bridge when they appear, making them visible to DDS and therefore recordable, while concurrently running `ros2 bag record`. All standard `ros2 bag record` arguments (e.g. `-e`) are passed through unchanged. It also works for standard ROS 2 topics.

`record_agnocast` works cluster-wide wherever a discovery agent runs (across IPC namespaces and ECUs on the same `ROS_DOMAIN_ID`); without a discovery agent, Agnocast-only topics cannot be recorded (they are silently missed).

When the discovery agent is running:

```bash
$ ros2 bag record_agnocast -o rosbag_output -e /my_*
[INFO 1781059372.244129306] [rosbag2_recorder]: Press SPACE for pausing/resuming (RecorderImpl() at ./src/rosbag2_transport/recorder.cpp:223)
[INFO 1781059372.244225399] [rosbag2_recorder]: Event publisher thread: Started (event_publisher_thread_main() at ./src/rosbag2_transport/recorder_event_notifier_impl.hpp:169)
[INFO 1781059372.245935596] [rosbag2_recorder]: Starting recording to 'rosbag_output' (record() at ./src/rosbag2_transport/recorder.cpp:303)
[INFO 1781059372.246716373] [rosbag2_recorder]: Listening for topics... (record() at ./src/rosbag2_transport/recorder.cpp:329)
[INFO 1781059372.246728940] [rosbag2_recorder]: Recording... (record() at ./src/rosbag2_transport/recorder.cpp:338)
[INFO 1781059372.246865915] [rosbag2_recorder]: Topics discovery started. (topics_discovery() at ./src/rosbag2_transport/recorder.cpp:629)
[INFO 1781059373.218892553] [rosbag2_recorder]: Subscribed to topic '/my_topic' (subscribe_topic() at ./src/rosbag2_transport/recorder.cpp:730)
[INFO 1781059379.681077909] [rosbag2_recorder]: Topics discovery stopped. (topics_discovery() at ./src/rosbag2_transport/recorder.cpp:671)
[INFO 1781059379.681410757] [rosbag2_recorder]: Pausing recording. (pause() at ./src/rosbag2_transport/recorder.cpp:547)
[INFO 1781059379.681530215] [rosbag2_cpp]: Writing remaining messages from cache to the bag. It may take a while (stop() at ./src/rosbag2_cpp/cache/cache_consumer.cpp:44)
[INFO 1781059379.684395117] [rosbag2_recorder]: Recording stopped (stop() at ./src/rosbag2_transport/recorder.cpp:281)
[INFO 1781059379.691063397] [rosbag2_recorder]: Event publisher thread: Exited (event_publisher_thread_main() at ./src/rosbag2_transport/recorder_event_notifier_impl.hpp:220)
$ ros2 bag info rosbag_output

Files:             rosbag_output_0.mcap
Bag size:          62.5 MiB
Storage id:        mcap
ROS Distro:        jazzy
Duration:          6.272831748s
Start:             Jun 10 2026 11:42:53.307379851 (1781059373.307379851)
End:               Jun 10 2026 11:42:59.580211599 (1781059379.580211599)
Messages:          64
Topic information: Topic: /my_topic | Type: agnocast_sample_interfaces/msg/DynamicSizeArray | Count: 64 | Serialization Format: cdr
Service:           0
Service information: 
```

Without a discovery agent, the command starts but records nothing for Agnocast-only topics (silently unobservable):

```bash
$ ros2 bag record_agnocast -o rosbag_output -e /my_*
[INFO 1781059455.269319259] [rosbag2_recorder]: Press SPACE for pausing/resuming (RecorderImpl() at ./src/rosbag2_transport/recorder.cpp:223)
[INFO 1781059455.269340122] [rosbag2_recorder]: Event publisher thread: Started (event_publisher_thread_main() at ./src/rosbag2_transport/recorder_event_notifier_impl.hpp:169)
[INFO 1781059455.271078152] [rosbag2_recorder]: Starting recording to 'rosbag_output' (record() at ./src/rosbag2_transport/recorder.cpp:303)
[INFO 1781059455.271900382] [rosbag2_recorder]: Listening for topics... (record() at ./src/rosbag2_transport/recorder.cpp:329)
[INFO 1781059455.271912956] [rosbag2_recorder]: Recording... (record() at ./src/rosbag2_transport/recorder.cpp:338)
[INFO 1781059455.271961053] [rosbag2_recorder]: Topics discovery started. (topics_discovery() at ./src/rosbag2_transport/recorder.cpp:629)
[INFO 1781059462.239325076] [rosbag2_recorder]: Topics discovery stopped. (topics_discovery() at ./src/rosbag2_transport/recorder.cpp:671)
[INFO 1781059462.239964342] [rosbag2_recorder]: Pausing recording. (pause() at ./src/rosbag2_transport/recorder.cpp:547)
[INFO 1781059462.240101000] [rosbag2_cpp]: Writing remaining messages from cache to the bag. It may take a while (stop() at ./src/rosbag2_cpp/cache/cache_consumer.cpp:44)
[INFO 1781059462.241862458] [rosbag2_recorder]: Recording stopped (stop() at ./src/rosbag2_transport/recorder.cpp:281)
[INFO 1781059462.244343500] [rosbag2_recorder]: Event publisher thread: Exited (event_publisher_thread_main() at ./src/rosbag2_transport/recorder_event_notifier_impl.hpp:220)
$ ros2 bag info rosbag_output

Files:             rosbag_output_0.mcap
Bag size:          1.8 KiB
Storage id:        mcap
ROS Distro:        jazzy
Duration:          0.000000000s
Start:             Apr 12 2262 08:47:16.854775807 (9223372036.854775807)
End:               Apr 12 2262 08:47:16.854775807 (9223372036.854775807)
Messages:          0
Topic information: 
Service:           0
Service information: 
```
