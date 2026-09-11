# Domain Bridge

The domain bridge connects Agnocast publishers and subscribers that run under **different `ROS_DOMAIN_ID`s** on the same machine — with **zero copy**. You register a rule for a topic, and Agnocast delivers that topic across the two domains by sharing the publisher's shared-memory message directly: no serialization, no relay process.

!!! warning "Registering rules is unsupported"
    The zero-copy cross-domain path is incomplete and not supported. Relay between domains with the external [ROS 2 `domain_bridge`](https://github.com/ros2/domain_bridge) node instead; the kernel module and `register_domain_bridge` both warn if you register a rule anyway.

    The rule file itself is still needed in that setup: the discovery agent reads it to bring up the Agnocast→ROS 2 bridge the external node relays from. See [Configuration](configuration.md).

!!! note "This is not the Agnocast–ROS 2 Bridge"
    The [**Bridge**](../migration-guide/bridge.md) connects Agnocast nodes to standard ROS 2 (RMW) nodes. The **domain bridge** on this page connects Agnocast nodes in one `ROS_DOMAIN_ID` to Agnocast nodes in another. They are independent features that happen to share the word "bridge".

## Domain isolation

Agnocast honors `ROS_DOMAIN_ID`: a publisher and a subscriber connect only when they share the same domain, matching ROS 2 semantics. (Earlier versions ignored the domain — every Agnocast endpoint in an IPC namespace connected to every other one.)

The domain bridge is the opt-in exception to that isolation. It re-connects **specific topics** across **one specific pair of domains**, and nothing else.

## Scope

- **Zero copy, same IPC namespace only.** The bridge shares the publisher's memory directly, so both domains must be on the same machine and in the same IPC namespace. Crossing ECUs or IPC namespaces is not covered here — that goes through the [Agnocast–ROS 2 Bridge](../migration-guide/bridge.md) over DDS, with a copy. See [Limitations](../index.md#limitations).
- **Agnocast ↔ Agnocast only**, in **performance mode**.

## Next steps

- [Configuration](configuration.md) — write a rule file and register it with `register_domain_bridge` before your nodes start.
- [Constraints & differences](constraints.md) — how the Agnocast domain bridge differs from the ROS 2 `domain_bridge`.
