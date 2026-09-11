# Configuring the Domain Bridge

You declare which topics to bridge, and between which domains, in a YAML file. Agnocast reuses the [ROS 2 `domain_bridge`](https://github.com/ros2/domain_bridge) file format, so existing config is familiar — Agnocast simply ignores the fields it does not support (see [Constraints](constraints.md)).

The discovery agent reads this file to bring up the Agnocast→ROS 2 bridge for a topic that crosses a domain. `register_domain_bridge` reads the same file and registers each rule with the kernel module — that path is [unsupported](index.md); run it once, **before the nodes that use those topics start**.

## Register rules before your nodes start

This is the most important rule to get right. A bridge rule can only be registered for a topic that has **no publisher or subscriber yet in either domain**. If an endpoint for the topic already exists when the rule is applied, the registration is **rejected**.

So `register_domain_bridge` must run — and register the rules — **before** the nodes that use those topics come up. The discovery agent does **not** register rules; this is a separate, one-time step.

!!! note "Boot-time loading is planned"
    For now you arrange this ordering yourself (see [Applying the config](#applying-the-config)). Loading bridge rules automatically at module-load time, before any application starts, is on the roadmap.

## File format

```yaml
# Default direction for every topic below. Per-topic values override these.
from_domain: 1
to_domain: 2

topics:
  # Bridge /chatter from domain 1 to domain 2 (uses the defaults above).
  chatter:

  # Override the direction for a single topic.
  image:
    from_domain: 3
    to_domain: 4
```

- The map key (`chatter`, `image`) is the topic name in the source domain. Add `remap:` to give it a different name in the destination domain.
- `from_domain` / `to_domain` are `ROS_DOMAIN_ID` values. Give them at the top level as defaults, per topic, or both.
- A topic with no resolved `from_domain` / `to_domain` (no default and no override) is skipped.
- The ROS 2 `type` field is accepted but ignored — matching is type-independent.

Each rule is one-directional (`from_domain` → `to_domain`). Set `bidirectional: true` on a topic to get the reverse direction as well. ROS 2's `reversed` option is ignored.

## Where the file goes

`AGNOCAST_DOMAIN_BRIDGE_CONFIG` names the files — one path, or several separated by `:`. Without it, `/etc/agnocast/domain_bridge.yaml` is read, followed by every `*.yaml` in `/etc/agnocast/domain_bridge.d/` in name order.

Splitting the rules across files is the same merge the ROS 2 `domain_bridge` node performs: `topics` from every file accumulate, while `from_domain` / `to_domain` stay local to the file that sets them. A later file only adds — it never overrides an earlier one, so two files that bridge the same topic and domain to different places are a configuration error, and the second rule is rejected. A file that cannot be read is reported and skipped, so the rest still apply.

The discovery agent is `execv`'d from an application process, so it never sees a `--config` argument: split its rules with the drop-in directory, or export the variable to your applications too.

## Applying the config

Run `register_domain_bridge`, pointing it at your files:

```bash
ros2 run ros2agnocast_discovery_agent register_domain_bridge
# reads the default locations above; override with --config a.yaml b.yaml
```

It registers every rule and exits. It is idempotent (re-running is safe) and exits non-zero if any rule is rejected — so a node that came up too early fails loudly instead of silently leaving a topic unbridged.

For production, run it from a boot step ordered **after** the kernel module is loaded and **before** your application nodes — for example a systemd one-shot. The `ros2agnocast_discovery_agent` package ships a reference unit (`systemd/agnocast-domain-bridge.service.example`) that shows this ordering.
