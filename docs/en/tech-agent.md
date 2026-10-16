# Letting automations and AI run GRISM: tokens, validation, trace, commit-confirm, capabilities and events

## Why

An AI or an automation running a data centre does not look at web pages; it looks at APIs and events. From where it stands, GRISM is the network's eyes and hands: it sees every packet (JA4, SNI, DNS, GTP, MEC sessions) and can change forwarding, capture or detour traffic at once. What was missing was not "can a program operate it" -- every operation had an API -- but three things that let a program operate it **safely**:

1. An identity that is not a person, with a boundary on what it may do and a record of what it did.
2. Knowing before a change goes wrong, and recovering by itself after it does.
3. Being told what happened without polling, and first knowing what this device can do at all.

Since 2026-10-16 (pywww 1.47.0 / 1.45.0, Studio 2.261.0) all three are there. The calls and their parameters are in the [web API note](/en/tech-web-api.md) under "Automation"; this is about the design.

## 1. API tokens: an identity for a program

Settings → Login authentication → API tokens. A token is made with a name, a level, an expiry in days and the addresses it may be used from; `gpx_…` is shown once and the device keeps only its SHA-256. Every request then carries `Authorization: Bearer <token>` -- no sign-in, no session to expire.

Deliberate limits:

- **A token's level tops out at config; it cannot be admin.** Firmware, accounts, the management IP and backup/restore stay with a person.
- It can be bound to addresses or networks; from anywhere else it is as if there were no token.
- A bad token -- unknown, expired, from the wrong address -- is answered 404 like a missing session, and **never** falls back to a cookie the same request may carry.
- The system log and the auth events record everything it does as `token:<name>`, apart from what people do.

## 2. Know before, recover after

**validate** -- the `<run>` is checked and not applied: well-formed XML, duplicate ids, the filters and outputs the chains refer to, ports that exist on this device, and every `<find>` name against the field catalogue this firmware was built with. The last one could not be known in advance before: a field the firmware does not know is **silently dropped**, leaving a filter looser than the one written. The catalogue now ships with the firmware (`etc/fields.json`, generated from the engine's `fc.c`), so validation checks against exactly what the engine accepts.

**trace** -- given an ingress port and a description of a packet (addresses, protocol, ports, VLAN, MACs, tunnel kind), the device works out from the configuration which chain it takes, which filters match and where it leaves. Filters the parameters can decide are decided; those that cannot be (session state, heartbeat, mapping tables, L7 content) fork the path, each branch recording what it assumed. `live=1` attaches each filter's current hit counters, so a program can set "the trace says it matches" against "it is matching now". A candidate configuration can be POSTed instead of the running one, so "will X still take the same path after this change" is asked before the submit.

To be honest about it: this is static analysis of `run.xml`, not the engine running a packet. It answers structural questions, not what deep parsing would say.

**commit-confirm** -- `submitxml` takes `confirm=<seconds>`. The device keeps the running file, applies the new one, and puts the old one back unless `confirm_submit` arrives in time. Routers have commit-confirmed for the change that locks you out; a packet broker can do the same, since one wrong chain can take the management network with it. One pending submit at a time; `rollback_submit` goes back without waiting. The pending state lives on the device's disk, not in memory, so a restart of the web service cannot forget to roll back. The Studio's Export page offers the same option, and shows a banner for a submit awaiting confirmation whoever made it -- a program included -- with confirm and roll-back buttons.

## 3. Know what it can do, be told what happens

**get_capabilities** -- one answer to "what is this": model, firmware and engine versions, product line, every port (with the LOOP ones marked), management addresses, storages, the filter fields it accepts, the table sizes and feature flags, memory, bypass pairs, and a map of the automation calls. A program meeting a device it has never seen reads this and knows how to write for it.

**Events** -- four kinds: `alert` (raised and cleared), `config` (submit, instant submit, configSet, confirm, rollback), `engine` (the packet engine's processes changed: a restart or a crash), `auth` (a token made or removed, a failed sign-in). Two ways to get them:

- **Webhooks** (Settings → Alerts → Webhooks, up to 8): each event is POSTed as JSON, `X-Grism-Event` naming the kind and, with a secret set, `X-Grism-Signature: sha256=<HMAC>` to verify. A target that cannot be reached is logged once, not again within five minutes.
- **`get_events?since=<id>`**: the device keeps the last 500; a program that missed a push catches up from its last `last_id`.

MEC events -- a UE attaching, a handover -- are not on this bus; they are many, and the engine's `mec` syslog target still carries them.

## Should an AI agent run on the device?

Not inference. The MIPS models give Linux about 530 MB and their cores belong to the packets; the arm64 models have more memory, but the CPU is spoken for just the same. Reasoning belongs off the box, one agent for dozens of devices, which is also where the whole-network view is. What belongs on the box is deterministic automation -- alert rules, the watcher, the idle sweep -- plus this interface, so that an outside agent can use it safely. An air-gapped or classified site that truly needs a local agent should get a sidecar with fixed resource limits, off the packet path, with a kill switch, and its changes still through commit-confirm -- a second phase.

## A typical agent loop

```
get_capabilities          → which ports, fields and features this device has
get_events / a webhook    → told of alerts, engine restarts, configuration changes
validate(candidate)       → zero problems before going on
trace(candidate, packet)  → the paths of the traffic that matters are unchanged
submitxml?confirm=60      → apply
watch counters, reachability → confirm_submit; otherwise do nothing, and it rolls back in 60 s
```

Every step is on record: who (which token), when, what, and the roll-backs the device did by itself.
