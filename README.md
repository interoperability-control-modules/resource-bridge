# VOLTTRON Resource Bridge

The Resource Bridge agent bridges one VOLTTRON device into another across protocols. In the
vocabulary of OpenFMB, an *adapter* connects a protocol-speaking device to the common bus; the
Message Bus Adapter does that between VOLTTRON and a foreign bus, and this agent does it between two
devices behind the platform driver: a SunSpec inverter and the IEEE 2030.5 server the platform
serves for it, a real DNP3 outstation and the Modbus unit a SCADA master polls, a 2030.5 utility
server's controls and the inverter they are meant for.

A *bridge* has a source and a target, both UAIs known to the presentation service. Every
publication of the source device (its `devices/<device>/all` polls and the `/multi` pushes a served
device receives from its peer) is turned into the target device's own points by the chain the
service chooses for the fields the message carries, and written to the target through the platform
driver's `set_multiple_points`. Telemetry into a served device and control into a client-role
device are the same operation; which points the target accepts is its interface's business, and
refused points come back as errors on the bridge's result topic.

The presentation service only answers mapping and transformation questions (`resolve` with
`direction='write'` and a `fields` hint). The policy lives here:

- change-only writes: a point is written when its value differs from what the target device is known
  to hold. That knowledge comes from the target's own publications (the agent subscribes to them),
  from this agent's successful writes until the device reports, and, when `stale_after` is set,
  from the driver's `last` RPC for points unknown or older than that many seconds. Comparing with
  the device rather than with the last write means a point another writer changed is re-asserted by
  the next source message (`write_unchanged` writes everything regardless);
- echo suppression: a publication that merely confirms a value this agent wrote to that device
  within `echo_window` seconds is not bridged back, so a pair of bridges between two devices does
  not loop;
- ordered curve batches: a chain output's `sequence` list is written batch by batch after the flat
  points, and a failed batch stops the sequence;
- per-point errors, never a raised exception mid-write, published with the chain's path and
  retention on `<result_topic_prefix>/<bridge key>`.

## Requirements

- `volttron-core` >= 2.0.0rc30
- the interoperability service running as `platform.presentation`
- the platform driver (`platform.driver`) owning the devices at both ends

## Configuration

```json
{
    "presentation_service": "platform.presentation",
    "driver": "platform.driver",
    "echo_window": 5.0,
    "stale_after": null,
    "bridges": [
        {"name": "pv_to_utility", "source": "site1/pv_inverter", "target": ["site1", "pv_sep2_server"]},
        {"name": "utility_to_pv", "source": "site1/pv_sep2_server", "target": "site1/pv_inverter",
         "points": ["DERControl_opModMaxLimW", "DERControl_opModFixedVar"]},
        {"name": "scada_view", "source": "site1/der", "target": "site1/der_modbus_view", "write_unchanged": true,
         "enabled": false}
    ]
}
```

`source` and `target` are UAIs (a list of segments, or one `/`-joined string). A plain UAI is a
*device* end: the source must lead to a canonical resource with a `publication_topic`, the target to
one with an `rpc_topic`. An end written as `{"uai": ..., "bus": true}` is a *bus* end: the UAI itself
is a topic on the VOLTTRON bus carrying messages in the UAI's alias format, an OpenFMB profile topic
such as `openfmb/solarmodule/SolarControlProfile/<mRID>`. A bus source is subscribed to (messages
published by this agent itself are ignored); a bus target is published to with the headers of
`icm.resources.openfmb.bus` (content type, protobuf message name, origin). This is how OpenFMB
lives on the VOLTTRON bus: the Message Bus Adapter in `relay` mode re-serialises those topics to and
from the foreign bus, and any OpenFMB-aware agent can publish or consume them.

```json
{"bridges": [
    {"name": "pv_reading", "source": "site1/pv_inverter",
     "target": {"uai": "openfmb/solarmodule/SolarReadingProfile/7d1a2b3c-0000-4000-8000-000000000001", "bus": true}},
    {"name": "pv_control", "source": {"uai": "openfmb/solarmodule/SolarControlProfile/7d1a2b3c-0000-4000-8000-000000000001", "bus": true},
     "target": "site1/pv_inverter"}
]}
```
`stale_after` (seconds, unset by default) makes the agent ask the driver's `last` RPC for a target
point whose last known value is older than that, or unknown, before deciding whether to write it;
unset, the agent relies on the target's publications and its own writes. The tracking is done by
`icm.resources.resource.ResourceWriter`, one per target device.

Served devices (a server the platform runs) must be declared with their own device format,
generated with the discovery tools' `--served` flag, so the write edges into their inputs exist.

### Ends declared locally

An end may carry a `definition` instead of being resolved through the presentation service: a canonical
(`data_format`, `publication_topic`, `rpc_topic`, `parameters`) or an alias (`target_format`, `codec`, the
`transform` stage list) in the form of `icm.resources.definitions`. The UAI then only names the end; the service
is not consulted for it, and the chain between the two ends still comes from the service.

```json
{"bridges": [{"name": "pv_to_mirror",
              "source": {"uai": ["site1", "pv"], "definition": {"kind": "canonical", "uai": ["site1", "pv"],
                         "data_format": "fake_pv", "publication_topic": "devices/site1/pv/all", "rpc_topic": "devices/site1/pv"}},
              "target": "site1/mirror"}]}
```

### Delivery

A bridge's `delivery` says when the source's arrivals become one message for the chain (`icm.resources.DeliveryPolicy`):
`each` (every arrival as it comes, the default), `complete` (once every field the chain reads has arrived since the
last write, or after `max_wait` seconds with what is present), `merge` (every arrival merged over the last known
values of the other fields, `stale_after` dropping old ones), `interval` (every `interval` seconds from the last known
values) or `trigger` (when one of `fields` changes). The fields the chain reads are learned from the chains the
service returns. Each result report then carries an `assembly` record naming the fields that were fresh, merged or
missing.

```json
{"bridges": [{"name": "pv_to_sep2", "source": "site1/pv", "target": "site1/pv_sep2_server",
              "delivery": {"kind": "complete", "max_wait": 5}}]}
```

## RPC

- `list_bridges()`: the configured bridges, their resolved ends and the chains used so far.
- `relay_message(bridge, message, headers=None)`: push one message through a bridge as if its source
  had published it.
- `write(uai, message, as_format=None)`: write a protocol message (in `as_format`, or the format
  of the outermost alias in `uai`) to the device the UAI leads to. Returns the write result with the
  chain's path and retention.

- `local_definitions()`: the definitions this agent declares on its bridge ends, as `[{bridge, end, uai, definition}]`,
  so a configuration tool can show them beside the service's resources.

## Tests

`pytest tests` runs the unit tests (VOLTTRON stubbed). `tests/test_e2e_matrix.py` runs the
interoperability service's end-to-end matrix with this agent doing the writes for every
device-to-device path, and once more with OpenFMB on the VOLTTRON bus (this agent's bus ends plus the
Message Bus Adapter in relay mode) carrying the OpenFMB paths; it needs the modular driver stack and the Modbus, DNP3 and 2030.5 proxies
importable and is skipped otherwise.

The `statistics` RPC reports, per bridge, messages received, transform errors, writes and publishes, points
written, skipped (the target already held the value) and refused, the times of the last message, write and
error, and the target points another writer has moved away from the intended value.
