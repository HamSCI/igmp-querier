# igmp-querier — Requirements Specification

**Status:** v0.1 baseline (retroactive). **Owner:** Michael Hauan (AC0G).
**Last reconciled against code:** igmp-querier `fdbecc3` (2026-06-25).
**Prefix:** `IGQ`.

> Application of [sigmond/docs/REQUIREMENTS-TEMPLATE.md](https://github.com/HamSCI/sigmond/blob/main/docs/REQUIREMENTS-TEMPLATE.md)
> to an **infrastructure mitigation**, not a sigmond client. igmp-querier has
> **no client-contract surface** — no `inventory`/`validate`/`deploy.toml`,
> and `smd` does not manage it (see §8.3, which is intentionally N/A). This
> doc is deliberately **short**: the component is one ~583-line stdlib file,
> no package, no tests, by design. Provenance tags: `[DOC]` documented ·
> `[CODE]` implicit-in-code · `[NEW]` surfaced by this review. Status:
> ✅ implemented · 🟡 partial/unverified · ⬜ planned.

## 1. Context & problem statement

A SigMonD station distributes radiod RTP IQ/audio over **IP multicast**. On a
LAN whose managed switch has **IGMP snooping enabled but no IGMP querier**
(common with prosumer gear — TP-Link Easy Smart, Netgear Plus — and any
segment without an L3 router doing IGMP), the switch learns a client's initial
Join, forwards multicast for a while, then **silently stops** once the snooping
entry expires (~260 s). Streams "drop out" with no error anywhere — the classic
IGMP-snooping silent failure documented in `sigmond/docs/networking.md`.

igmp-querier mitigates this by acting as the missing querier: it sends periodic
**IGMPv2 General Queries** (RFC 2236) to `224.0.0.1`, which prompt hosts to
re-report group memberships and keep the switch's snooping table warm. It does
RFC 2236 querier **election** (lowest IP wins) so it is safe to run on more than
one host. It is a host-level network daemon — not a sigmond-managed client and
not a multicast router.

## 2. Goals & objectives

- Keep radiod multicast flowing on a snooping-without-querier LAN — no stream
  drop-out past the switch's ~260 s snooping timeout.
- Be a correct, RFC-compliant querier (v2 General Queries, RFC 2113 Router
  Alert, RFC 2236 election, RFC 3376 jitter) so it coexists with other queriers.
- Run unattended and self-heal across socket errors and interface IP changes.
- Stay a **single-file, stdlib-only, drop-on-a-host** mitigation — minimal
  footprint, minimal privilege.

## 3. Non-goals / out of scope

- **Multicast routing / forwarding.** It keeps snooping state warm; it does not
  route traffic. (Owner: the network's L3 device, if any.)
- **Being a sigmond client.** No contract surface, no `deploy.toml`; `smd` does
  not start/stop/inventory it — `systemctl` does. (Owner: host operator.)
- **Acting as an IGMPv3 querier.** It sends v2 General Queries (it only *parses*
  v3 for election). v3 host reports remain visible to snooping switches.
- **Fixing snooping config on the switch** or replacing a switch/router that
  could itself be the querier — preferred fixes per `networking.md` §`lan-needs-querier`.

## 4. Stakeholders & actors

Host operator (installs/selects interface) · the IGMP-snooping switch (consumer
of the queries) · multicast receivers/hosts on the segment (re-report on query) ·
`radiod` and its RTP clients (the protected traffic; indirect) · peer
igmp-queriers on the segment (RFC 2236 election) · `systemd` (lifecycle,
hardening) · sigmond `networking.md` (points operators here; does **not** manage it).

## 5. Assumptions & constraints

- `IGQ-C-001` `[DOC]` ✅ Linux with raw-socket support; needs **root or
  `CAP_NET_RAW`** (raw `IPPROTO_IGMP` socket).
- `IGQ-C-002` `[DOC]` ✅ **stdlib-only**, Python 3.6+; no external packages, no
  `pyproject.toml`, no venv — single file at `/usr/local/bin/igmp_querier.py`.
- `IGQ-C-003` `[CODE]` ✅ Must run on the **L2 segment that carries radiod
  multicast** (the default-route LAN iface), not a VPN/virtual iface; the
  installer auto-selects the default-route iface in `--yes` mode.
- `IGQ-C-004` `[DOC]` ✅ Intentionally **flat / single-file**; keep that shape
  (CLAUDE.md) — packaging would inflate the "drop one file and run" story.
- `IGQ-C-005` `[CODE]` ✅ One sender per segment is the common case; election
  makes multiple instances safe but is not the primary mode.

## 6. Functional requirements

### 6.1 Querying
- `IGQ-F-001` `[DOC]` ✅ SHALL send **IGMPv2 General Queries** (type `0x11`,
  group `0.0.0.0`, max-resp 100 = 10 s) to `224.0.0.1` at a configurable
  interval (default **60 s**, not RFC-125, tighter for home LANs).
- `IGQ-F-002` `[DOC]` ✅ SHALL compute the IGMP checksum per RFC 1071 and attach
  the **RFC 2113 Router Alert** IP option to each query.
- `IGQ-F-003` `[DOC]` ✅ SHALL send a **startup burst** of 3 queries 5 s apart
  (RFC 2236) so switches learn topology quickly, then settle to the interval.
- `IGQ-F-004` `[DOC]` ✅ SHALL apply **up to 25 % random jitter** (RFC 3376) to
  the interval to avoid querier/host synchronization.
- `IGQ-F-005` `[CODE]` ✅ Queries SHALL use multicast TTL=1 (not forwarded off the
  local segment).

### 6.2 Election (RFC 2236)
- `IGQ-F-010` `[DOC]` ✅ SHALL run a listener that receives IGMP queries on the
  interface and parses v1/v2/v3 for election purposes.
- `IGQ-F-011` `[DOC]` ✅ On a query from a **lower** source IP, SHALL back off
  (become passive); on a **higher** IP, SHALL continue as master — lowest IP wins.
- `IGQ-F-012` `[DOC]` ✅ When a winning competitor is silent past `--timeout`
  (default 255 s), SHALL re-assume master and resume querying.
- `IGQ-F-013` `[CODE]` ✅ The raw socket SHALL **not** `bind()` to the unicast
  IP (commit `6db4cf8`): binding filters out `224.0.0.1`-dest queries and
  silently breaks election; interface scoping uses `SO_BINDTODEVICE` instead.

### 6.3 Resilience
- `IGQ-F-020` `[CODE]` ✅ SHALL detect interface **IP change**, update its
  election identity, and rebuild the socket without operator action.
- `IGQ-F-021` `[CODE]` ✅ SHALL recover from socket / send errors: after
  `MAX_CONSECUTIVE_ERRORS` (10) it rebuilds the socket and restarts the listener.
- `IGQ-F-022` `[CODE]` ✅ SHALL refuse to start if the interface has no valid
  unicast IPv4 (exit non-zero), so systemd `Restart=always` retries cleanly.

### 6.4 Control & lifecycle
- `IGQ-F-030` `[DOC]` ✅ SHALL accept CLI `--interface` (required),
  `--query-interval`, `--timeout`, `--verbose`.
- `IGQ-F-031` `[DOC]` ✅ SHALL shut down gracefully on SIGTERM/SIGINT, join the
  listener, and log final stats (queries sent, elections lost).
- `IGQ-F-032` `[DOC]` ✅ `install.sh` SHALL detect interfaces, select one
  (interactive, single-iface auto, `IGMP_INTERFACE=`, or `--yes` default-route),
  install the file + hardened unit, and enable/start it; `uninstall.sh` reverses it.

## 7. Quality / non-functional requirements

- `IGQ-Q-001` `[DOC]` ✅ SHALL run under **least privilege**: systemd unit
  `User=nobody`, `CapabilityBoundingSet=CAP_NET_RAW`, `AmbientCapabilities=CAP_NET_RAW`,
  `NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp`, `MemoryDenyWriteExecute`.
- `IGQ-Q-002` `[CODE]` ✅ Shared state SHALL be thread-safe (`QuerierState`
  lock-guarded) across sender and listener threads.
- `IGQ-Q-003` `[DOC]` ✅ SHALL be observable via journald only (no log files);
  `SyslogIdentifier=igmp-querier`, `journalctl -u igmp-querier`.
- `IGQ-Q-004` `[DOC]` ✅ SHALL auto-restart on crash (`Restart=always`,
  `RestartSec=5`) and survive interface flaps via its own recovery path.
- `IGQ-Q-005` `[NEW]` 🟡 Resource footprint SHALL be negligible (one sleep-bound
  thread + one recvfrom-bound thread); asserted, not measured.

## 8. External interfaces

### 8.1 Inputs
- **CLI:** `-i/--interface` (required), `-q/--query-interval` (60), `-t/--timeout`
  (255), `-v/--verbose`.
- **Install env:** `IGMP_INTERFACE=<name>`, `--yes` (non-interactive install).
- **Network:** inbound IGMP queries on the interface (election input).
- No config file; the interface is baked into the systemd `ExecStart` at install.

### 8.2 Outputs
- **On the wire:** IGMPv2 General Queries to `224.0.0.1` (RA option, TTL 1).
- **Logs:** journald (startup banner, election transitions, recovery, shutdown
  stats). No sink writes, no files, no upload targets.

### 8.3 Contracts / APIs — **N/A (intentional)**
- `IGQ-I-001` `[DOC]` ✅ igmp-querier has **no client-contract surface** and
  this is by design: it is an infrastructure mitigation, not a SigMonD client.
  It does **not** implement `inventory`/`validate`/`version --json`, ships **no**
  `deploy.toml`, and is **not** discovered or lifecycle-managed by `smd`
  (CLIENT-CONTRACT.md does not apply). Lifecycle is plain `systemctl`; sigmond's
  only relationship is documentary — `sigmond/docs/networking.md` points
  operators here when `smd admin diag net` reports `lan-needs-querier`.

## 9. Data requirements

None. No persistent store, no schema, no retention, no data products. In-memory
counters only (`queries_sent`, `elections_lost`, `am_querier`), logged at exit.

## 10. Dependencies & development sequence

**Runtime deps:** Linux kernel raw-socket + `CAP_NET_RAW`; Python 3.6+ stdlib
(`socket`, `struct`, `fcntl`, `threading`, `signal`). No third-party libs, no
sigmond, no radiod *dependency* (radiod is the beneficiary, not a requirement —
the daemon runs standalone).

**Sequence (recovered as intent):** v2 General-Query sender → RFC 2236 election
listener → the no-`bind()` election fix (`6db4cf8`) → self-healing (IP-change /
socket recovery) → hardened systemd unit + interface-aware installer. Future
work is gated behind "is the single-file shape still worth keeping?" — kept.

## 11. Acceptance criteria & verification

- Querying → `tcpdump -i <iface> igmp` shows periodic `igmp query v2` from this
  host's IP; ka9q-radio streams stay up past the switch's ~260 s timeout.
- Election → with a second querier present, the higher-IP instance logs
  "LOST to lower IP" and stops sending; killing the master, the survivor takes
  over within `--timeout`.
- Service health → `systemctl status igmp-querier` active; journal shows the
  startup banner and burst.
- Recovery → bouncing the interface IP triggers a logged rebuild and queries
  resume (manual check; no automated test — see `IGQ-Q-006`).

## 12. Risks & open questions

- `IGQ-Q-006` `[NEW]` ⬜ **No test suite** (intentional today). Election, the
  no-`bind()` invariant (`IGQ-F-013`), checksum (`IGQ-F-002`), and IP-change
  recovery (`IGQ-F-020`) are unit-testable in principle; their correctness is
  currently verified only by manual `tcpdump`/field observation. A minimal
  pure-function test (checksum, packet build, `ip_to_int`, election compare)
  would be low-cost insurance. *(Surfaced by this review.)*
- `IGQ-F-040` `[NEW]` ⬜ **No `--version` / identity output.** There is no way
  to confirm which build is running beyond reading the file mtime; a one-line
  `--version` would aid fleet reasoning. *(Surfaced by this review.)*
- `IGQ-Q-007` `[NEW]` ⬜ **Footprint unmeasured** (`IGQ-Q-005`): CPU/memory
  asserted negligible but never profiled under sustained operation.
- Doc/operability note: README still cites the standalone
  `git clone github.com/HamSCI/igmp-querier`; on a sigmond host the repo already
  lives at `/opt/git/sigmond/igmp-querier` — harmless, but two install paths.

## 13. Traceability

| Requirement | #18 issue | Verification | PSWS #6 |
|---|---|---|---|
| IGQ-F-001 (v2 queries) | — | `tcpdump igmp` shows v2 query | #6 (multicast readiness) |
| IGQ-F-011/012 (election) | — | two-instance master/takeover test | — |
| IGQ-F-013 (no unicast bind) | — | election works on multi-host LAN (commit 6db4cf8) | — |
| IGQ-Q-001 (least privilege) | — | unit runs as nobody w/ CAP_NET_RAW | — |
| IGQ-Q-006 (no tests) | *(new — file)* | add pure-function unit tests | — |
| IGQ-F-040 (no --version) | *(new — file)* | add `--version` flag | — |
| IGQ-Q-007 (footprint) | *(new — file)* | profile under load | — |

*New rows (IGQ-Q-006, IGQ-F-040, IGQ-Q-007) are the review's surfaced gaps —
all consistent with the deliberately-minimal stub maturity.*
