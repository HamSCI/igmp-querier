# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

**igmp-querier** is a small, single-file Python daemon that acts as
an **IGMPv2 Querier** (RFC 2236) for networks where IGMP snooping is
enabled on switches but no active Querier exists.

Without an active Querier, IGMP-snooping switches eventually treat
multicast groups as expired and silently stop forwarding multicast
traffic — including the RTP streams ka9q-radio uses. This daemon
sends periodic General Queries to keep group memberships alive.

Part of the HamSCI sigmond suite — see `/opt/git/sigmond/sigmond/CLAUDE.md`
(orchestrator) and `/opt/git/sigmond/CLAUDE.md` (umbrella) for
cross-repo context. It is an **infrastructure mitigation**, not a
sigmond client (no contract surface, no `deploy.toml`).

## Authors

- Michael James Hauan — adapted for KA9Q-radio support.
- Repo: https://github.com/HamSCI/igmp-querier

## Layout

This repo is intentionally **flat — no `pyproject.toml`, no `src/`
tree, no test suite**. The whole daemon is one file:

```
igmp_querier.py        # 583 lines — daemon (raw socket sender + RFC 2236 election)
igmp-querier.service   # systemd unit (User=nobody, CAP_NET_RAW only)
install.sh             # /usr/local/bin/ install + systemd enable
uninstall.sh
README.md              # network-topology context, deployment guidance, RFC notes
```

When editing, keep the single-file shape unless there's a strong
reason — the deployment story is "drop one Python file on a host with
`CAP_NET_RAW` and run it." Pulling in `pyproject.toml` / packaging
would inflate that.

## Key design facts

- **Raw socket sender on `IPPROTO_RAW`**, not the high-level socket
  API. The daemon builds its own IP + IGMP packets to include the
  RFC 2113 Router Alert option — required for routers to process IGMP
  packets correctly.
- **RFC 2236 election (lowest IP wins).** Multiple queriers on the
  same segment elect a master; the others become listeners. The fix
  in commit `6db4cf8` ensures the raw socket does **not** `bind()`
  to a unicast IP — that would silently break the election by
  filtering out other queriers' packets.
- **Query jitter** (RFC 3376 §4.1.1): up to 25 % of the query
  interval, so multiple queriers don't synchronise on the same wire
  tick.
- **Default query interval is 60 s**, not the RFC-default 125 s.
  Tighter for home LANs where the failure mode (multicast falling
  silent) is more painful than the extra IGMP traffic.
- **Startup behaviour** (RFC 2236): 3 rapid queries 5 s apart so
  switches learn the multicast topology quickly.
- **Self-healing**: detects IP changes, socket errors, and
  consecutive-error thresholds; recovers without operator
  intervention.
- **Hardened systemd unit**: `User=nobody`, `CapabilityBoundingSet=
  CAP_NET_RAW`, `ProtectSystem=strict`, `PrivateTmp=true`,
  `MemoryDenyWriteExecute=true`, etc. — minimum privilege.

## Commands

```bash
# Install (interactive: prompts for interface)
sudo ./install.sh

# Non-interactive
sudo IGMP_INTERFACE=enp1s0 ./install.sh --yes

# Run by hand for debugging
sudo python3 igmp_querier.py --interface enp1s0

# Uninstall
sudo ./uninstall.sh

# Service ops once installed
systemctl status igmp-querier
journalctl -u igmp-querier -f
```

## When to deploy this

Run it on **exactly one host per multicast segment** (the election
handles multiple-querier cases, but a single sender is the common
case).

Symptoms that justify deployment:

- ka9q-radio multicast streams arrive at one host directly connected
  to radiod (e.g. `tcpdump` sees them) but not at hosts behind a
  switch.
- Switch supports IGMP snooping but its `show ip igmp snooping
  querier` output is empty / says "none."
- Streams work briefly after a switch reboot and then go silent —
  classic snooping-timeout symptom.

`sigmond/docs/networking.md` has the deeper diagnostic walkthrough.

## What this project is NOT

- Not a Python package (no `pyproject.toml`, no installable wheel —
  intentionally).
- Not a sigmond client (no inventory / validate / deploy.toml).
  Sigmond's `smd` doesn't manage it; `systemctl` does.
- Not a router or a multicast-routing daemon — it doesn't route
  traffic, it just keeps switches' snooping state warm.
- Not IGMPv3-aware as a Querier — it sends v2 General Queries.
  IGMPv3 host reports are still seen by snooping switches when v2
  queries are sent.
