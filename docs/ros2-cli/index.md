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
- **Agnocast version** — whether `ros2agnocast` provides a dedicated Agnocast-aware verb (e.g. `ros2 topic list_agnocast`, `ros2 agnocast generate-bridge-plugins`). ✓ means a variant exists; ✗ means no variant exists today. Most Agnocast-aware verbs are **local-scope only** — they inspect shared memory and the Agnocast kernel module on the machine they run on, so they cannot discover Agnocast endpoints living on other hosts. The verb-level tables below mark this explicitly.
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
| `ros2 bag` | ✓ | ✗ | - | Works for bridged topics via the DDS side of the bridge. Purely Agnocast-only topics (without a bridge) are not captured. |
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
| `ros2 topic` | ⚠ | ✓ (`list_agnocast`, `info_agnocast`) | - | `as-is` does not see pure Agnocast-only topics at all, and for bridged topics it only reports the DDS side (no Agnocast pub/sub count, no `(Agnocast enabled)` marking). Use the `_agnocast` verbs to include Agnocast topics and labels. See §3. |
| `ros2 agnocast` | N/A | ✓ (new top-level command) | - | Provided by `ros2agnocast` for Agnocast-specific operations (`version`, `generate-bridge-plugins`). |

### 2. `ros2 node` verbs

| Verb | Works as-is | Agnocast version | Scope of Agnocast verb | Planned | Notes |
|------|:-----------:|:----------------:|:----------------------:|:-------:|-------|
| `ros2 node list` | ⚠ | ✓ `list_agnocast` | Local host only | - | `as-is` does not show pure `agnocast::Node` instances; only `rclcpp::Node`-based nodes are listed. `list_agnocast` merges that output with Agnocast-derived nodes discovered on the **local host's** shared memory. Agnocast nodes running on a remote host will not appear in the Agnocast-marked entries. Supports `-a`, `-c`, and `-d` (debug to include internal `agnocast_bridge_node_*`). |
| `ros2 node info` | ⚠ | ✓ `info_agnocast` | Local host only | - | `as-is` cannot target a pure `agnocast::Node` (it is not in `ros2 node list`); even for visible `rclcpp::Node`-based nodes, Agnocast publishers/subscribers are not labelled. `info_agnocast` augments the standard output with Agnocast publishers/subscribers and bridge status labels read from the local Agnocast state. For a node running on a remote host, run `info_agnocast` on that host. |

### 3. `ros2 topic` verbs

| Verb | Works as-is | Agnocast version | Scope of Agnocast verb | Planned | Notes |
|------|:-----------:|:----------------:|:----------------------:|:-------:|-------|
| `ros2 topic list` | ⚠ | ✓ `list_agnocast` | Local host only | - | `as-is` only lists topics that have DDS endpoints; pure Agnocast-only topics are missing, and bridged topics appear without any indication that an Agnocast side exists. `list_agnocast` adds Agnocast-only topics and tags Agnocast-touched topics with `(Agnocast enabled)` / `(Agnocast enabled, bridged)`. Local-host scope: Agnocast topics whose publishers and subscribers all live on a remote host are not discovered. See [Topic List](#topic-list) below. |
| `ros2 topic info` | ⚠ | ✓ `info_agnocast` | Local host only | - | `as-is` cannot target a pure Agnocast-only topic (it is not in `ros2 topic list`); for bridged topics it shows only DDS publishers/subscribers. `info_agnocast` reports Agnocast publisher/subscriber counts and QoS observed on the local host alongside the DDS side. Endpoints on other hosts only show up if they reach this host through the Agnocast↔ROS 2 bridge (DDS side). Supports `-v` and `-d`. See [Topic Info](#topic-info) below. |
| `ros2 topic echo` | ⚠ | ✗ | — | TBD | Works when the bridge is active, but the message type must be specified explicitly, e.g. `ros2 topic echo /my_topic agnocast_sample_interfaces/msg/DynamicSizeArray`. |
| `ros2 topic pub` | ⚠ | ✗ | — | TBD | Publishes via DDS, so messages reach Agnocast subscribers only via the bridge. The `--wait-matching-subscriptions` option does **not** work correctly for Agnocast subscribers — DDS discovery counts only the bridge-side DDS subscription (if any), not the Agnocast subscribers behind it. |
| `ros2 topic hz` | ✗ | ✗ | — | TBD | Subscribes via DDS to measure publish rate. For Agnocast publishers the data path is shared memory and DDS sees no messages, so no rate is reported (even when the bridge is active, the DDS rate observed by `hz` does not necessarily reflect the Agnocast publisher's rate). |
| `ros2 topic bw` | ✗ | ✗ | — | TBD | Requires subscribing to the topic. |
| `ros2 topic delay` | ✗ | ✗ | — | TBD | Requires subscribing to the topic. |
| `ros2 topic type` | ⚠ | ✗ | — | No | Resolves a topic name to its message type via the DDS graph. Works for bridged topics (the bridge's DDS endpoints carry the type). For pure Agnocast-only topics, no Agnocast version is planned: Agnocast does not retain a runtime type name (only a hash) per topic, so a name→type lookup is not possible in the general case. `ros2 topic info_agnocast <topic>` falls back to `<UNKNOWN>` for the same reason. |
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

`ros2 bag` records DDS traffic; Agnocast-only topics live in shared memory and are not captured unless the Agnocast↔ROS 2 bridge is active.

| Verb | Works as-is | Agnocast version | Planned | Notes |
|------|:-----------:|:----------------:|:-------:|-------|
| `ros2 bag record` | ✓ | ✗ | - | Works via the bridge. Agnocast-only topics without a bridge are not captured. |
| `ros2 bag play` | ✓ | ✗ | - | Works via the bridge. Playback publishes via DDS; reaches Agnocast only if a bridge exists. |
| `ros2 bag info` | ✓ | ✗ | - | Operates on a stored bag file, independent of transport. |
| `ros2 bag list` | ✓ | ✗ | - | Same as above. |
| `ros2 bag convert` | ✓ | ✗ | - | Same as above. |
| `ros2 bag reindex` | ✓ | ✗ | - | Same as above. |
| `ros2 bag burst` | ✓ | ✗ | - | Same as `play`. |

### 10. `ros2 agnocast` verbs

`ros2agnocast` registers a top-level `ros2 agnocast` command with the following verbs (see [`src/ros2agnocast/setup.py`](https://github.com/autowarefoundation/agnocast/blob/main/src/ros2agnocast/setup.py)).

| Verb | Scope | Purpose |
|------|:-----:|---------|
| `ros2 agnocast --version` / `-v` | Local host only | Print version information for Agnocast components installed on the host where the command is run (userland libraries and the loaded kernel module). |
| `ros2 agnocast version` | Local host only | Same as `--version` (verb form). |
| `ros2 agnocast generate-bridge-plugins` | Build-time | Generate a ROS 2 bridge plugin package for user message types. Operates on local source/install trees, not on any running system. |

---

## How to use ros2 command for Agnocast

Currently, Agnocast supports the following `ros2` commands:

- `ros2 topic list`
- `ros2 topic info /topic_name`
- `ros2 node list`
- `ros2 node info /node_name`

!!! warning "Local-host scope"
    Each of the Agnocast-specific verbs below (`list_agnocast`, `info_agnocast`) inspects the Agnocast state on **the host where the command runs**: the kernel module, `/dev/agnocast`, and shared-memory segments. To inspect Agnocast endpoints on another host, run the same command on that host. The DDS-side portion of each command's output is still cluster-wide, as usual.

### Topic List

To list all topics including Agnocast, use `ros2 topic list_agnocast`.

The `(Agnocast enabled)` marking comes from the local host's Agnocast state. ROS 2 (DDS-side) topics are still discovered cluster-wide, but a topic whose Agnocast endpoints all live on a remote host will appear as a plain ROS 2 topic from this command's perspective.

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
| rclcpp::publisher | rclcpp::subscription | off / standard / performance | /my_topic |
| agnocast::publisher | rclcpp::subscription | off | /my_topic (WARN: Agnocast and ROS 2 endpoints exist but bridge is not active) |
| agnocast::publisher | rclcpp::subscription | standard / performance | /my_topic (Agnocast enabled, bridged) |
| rclcpp::publisher | agnocast::subscription | off | /my_topic (WARN: Agnocast and ROS 2 endpoints exist but bridge is not active) |
| rclcpp::publisher | agnocast::subscription | standard / performance | /my_topic (Agnocast enabled, bridged) |
| agnocast::publisher | agnocast::subscription | off / standard / performance | /my_topic (Agnocast enabled) |
| agnocast::publisher | non | off / standard / performance | /my_topic (Agnocast enabled) |
| non | agnocast::subscription | off / standard / performance | /my_topic (Agnocast enabled) |

#### Notes

- If an Agnocast topic and a ROS 2 topic share the same name but have different message types, the status will still be displayed as (Agnocast enabled, bridged). However, in this case, the actual communication will not be established.

### Topic Info

To show the topic info including Agnocast, use `ros2 topic info_agnocast /topic_name`.

The Agnocast publisher/subscriber counts and per-endpoint details below are read from the **local** Agnocast state. Endpoints on other hosts only appear if they reach this host through the Agnocast↔ROS 2 bridge (and then they show up on the DDS side).

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

### Node List

To list all nodes including those implemented with Agnocast, use `ros2 node list_agnocast`.

Only `agnocast::Node` instances on the **local host** are augmented into the list. Run the command on each host to enumerate Agnocast nodes there.

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

Agnocast publisher/subscriber details for the node are taken from the **local** Agnocast state. If the target node is running on a different host, run `info_agnocast` on that host to see its Agnocast endpoints; otherwise only the DDS-visible portion of the node's interface is reported.

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
