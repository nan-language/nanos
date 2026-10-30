# Network

This chapter defines the network stack and its protocol suite, the network configuration and its reconciliation, address configuration by DHCPv4 and SLAAC, name resolution, the egress gate with the `Egress` capability and its flows, ingress through the `Listen` capability, the origin labels of received data, provisional configuration changes, the network transports of the link with the Noise_IK handshake of TCP, and the `net` code family.

## The network stack

`net` is the [base instance](20-base.md#base-instances) that implements networking. It runs while the `nic` role holds a device, a vsock device included ([volumes and roles](15-devices.md#volumes-and-roles)), and each `Nic` device holding that role is an **interface** named by the last segment of its driver's moniker (`virtio-net0`). It implements exactly these protocols:

| Layer | Protocols |
| --- | --- |
| link | Ethernet II (IEEE 802.3); frames with 802.1Q tags are dropped |
| neighbours | ARP (RFC 826) with address conflict detection (RFC 5227); neighbour discovery (RFC 4861) with duplicate address detection (RFC 4862); MLDv2 (RFC 3810) for its solicited-node groups |
| network | IPv4 (RFC 791), IPv6 (RFC 8200), ICMPv4 (RFC 792), ICMPv6 (RFC 4443), path MTU discovery (RFC 1191, RFC 8201) |
| transport | UDP (RFC 768); TCP (RFC 9293) with SACK (RFC 2018), window scaling and timestamps (RFC 7323), CUBIC (RFC 9438), an initial window of 10 segments (RFC 6928) and SYN cookies (RFC 4987); virtio-vsock streams (Virtio 1.2 §5.10) |
| configuration | DHCPv4 (RFC 2131, RFC 2132, RFC 3442), SLAAC (RFC 4862) with stable interface identifiers (RFC 7217) and RDNSS (RFC 8106) |
| names | a DNS stub resolver (RFC 1035, RFC 3596, RFC 7766) |

- `net` never forwards a packet between interfaces, never sends IPv4 options, and has no loopback interface: instances reach each other through [ports](07-communication.md#ports).
- It reassembles IPv4 and IPv6 fragments up to 64 KiB per packet and 1 MiB in all, dropping a packet still incomplete after 30 s.
- Every frame arrives `Untrusted` from its driver ([NaN 15 §Sources](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#sources)), and every parser of `net` is a validator over `Untrusted` views ([NaN 15 §Validators](https://github.com/nan-language/nan/blob/main/docs/spec/15-untrusted.md#validators)).
- `net` runs one keyed task per `Nic` queue, on the CPU the queue's vector targets (requires [N8](22-nan.md#n8)); a TCP connection is processed on the queue its flow hashes to. Received buffers pass from the driver to their consumer without a copy inside the kernel image's bundle, and as encodings to other bundles ([transfer](07-communication.md#transfer)). Polling under load is the `Nic` driver rule ([Nic](15-devices.md#nic)).
- TLS 1.3 and Noise are libraries their clients link; `net` carries their bytes.
- `net` is `checkpointed` ([durability and checkpoints](06-execution.md#durability)). Its flows are part of its state and its keyed tasks only move frames, so an Env rebuild keeps every flow; a restart keeps its leases and identifier secret and resets every flow.
- Without a flow, a lease or an address probe in progress, `net` arms no timer.

## Configuration

Desired state is the configuration record of `net`, `/sys/config/net`, of type `NetConfig`. A write to it activates `net` ([plans](14-changes.md#plans-and-steps)), which reconciles `/live/net` to it.

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `interfaces` | map of interface name to `Interface` | empty | per-interface settings; an entry for an absent interface applies when the interface appears |
| `default` | `Interface` | `enabled: true`, `ipv4: Ipv4Mode.Dhcp`, `ipv6: Ipv6Mode.Slaac` | the settings of an interface without an entry |
| `routes` | list of `Route` | empty | static routes |
| `dns` | `DnsConfig` | `servers: []`, `learned: true` | resolvers ([DNS](#dns)) |

| Record | Fields |
| --- | --- |
| `Interface` | `enabled: bool`; `mtu_bytes: Option[u32]` (else the device's); `ipv4: Ipv4Mode`, one of `Dhcp`, `Static(address: text, gateway: Option[text])` and `Off`; `ipv6: Ipv6Mode`, one of `Slaac`, `Static(address: text, gateway: Option[text])` and `Off` |
| `Route` | `destination: text`, `gateway: text`, `interface: Option[text]`, `metric: u32` (by default 100) |
| `DnsConfig` | `servers: list of text`, `learned: bool` |

- `Ipv6Mode.Off` disables IPv6 on the interface, link-local address included; `Slaac` and `Static` keep the link-local address.
- A route is chosen by longest prefix, then lowest metric, then static before learned. Learned routes (DHCP routers and classless routes, router advertisements) have metric 1,000.
- Addresses are texts in their RFC forms, with a prefix length for `address` and `destination` (`10.0.0.9/24`, `2001:db8::/32`). Another text, a static address in a loopback, multicast or unspecified range, or a prefix longer than the address fails the write with `schema.invalid` ([validity](11-store.md#declared-types-and-validity)).

`/live/net` holds a `LiveNet`:

| Field | Type | Meaning |
| --- | --- | --- |
| `interfaces` | `mem::List[LiveInterface]` | each interface |
| `routes` | `mem::List[LiveRoute]` | the routes in use |
| `dns` | `mem::List[text::String]` | the DNS servers in use |
| `status` | `ReconcileStatus` | the configuration's status, `failed` with an interface's code while that interface's status is `failed` |
| `as_of` | `u64` | as in [/live](13-tree.md#live) |

| Record | Fields |
| --- | --- |
| `LiveInterface` | `name: text::String`; `link: LinkState` ([Nic](15-devices.md#nic)); `link_address: mem::Bytes`, the device's `NicInfo.address`; `mtu_bytes: u32`, the MTU in use; `addresses: mem::List[LiveAddress]`; `status: ReconcileStatus` |
| `LiveAddress` | `address: text::String`, with its prefix length as in `Interface`; `origin: AddressOrigin`, one of `Static`, `Dhcp`, `Slaac` and `LinkLocal`; `state: AddressState`, one of `Tentative`, `Preferred`, `Deprecated` and `Failed`; `expires: Option[WallTime]`, none for an address without a lifetime |
| `LiveRoute` | `route: Route`; `origin: AddressOrigin` |

Each status is a `ReconcileStatus` ([desired and observed state](13-tree.md#desired-and-observed-state)). Open flows are `LiveFlow` records at `/live/net/flows/<id>` ([egress gate](#egress-gate)).

`net` serves one operation:

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `configure` | `config: NetConfig` | — | `schema.invalid` | `recorded` |

`configure` writes `/sys/config/net`.

## Address configuration

DHCPv4 follows RFC 2131 with these choices.

- The client identifier (option 61) is type 1 with the interface's MAC address; no host name is sent. The parameter request list asks for options 1, 3, 6, 26 and 121. Classless routes (option 121) replace the router option when present (RFC 3442). An interface MTU (option 26) applies when it lies between 576 bytes and the device's MTU.
- Before using an offered address, `net` probes it with ARP (RFC 5227); a conflict sends DHCPDECLINE, emits `net.address-conflict` and restarts discovery after 10 s.
- Retransmissions back off as RFC 2131 §4.1 states, from 4 s to 64 s. After 5 unanswered attempts the interface's status is `failed(net.no-lease)`, and `net` retries every 10 min and at each link change.
- Renewal is at T1 and rebinding at T2, from options 58 and 59, else 0.5 and 0.875 of the lease. At expiry the address and its routes are removed (`net.address-removed`) and discovery restarts.
- `net` keeps each interface's last lease, which it requests again when the link comes up.

SLAAC follows RFC 4862 with these choices.

- Interface identifiers are stable and opaque (RFC 7217), derived from a 128-bit secret that `net` generates at its first start and keeps in its checkpoint. There are no temporary addresses.
- Duplicate address detection sends one neighbour solicitation and waits 1 s; a duplicate marks the address `Failed` and emits `net.address-conflict`, and the identifier is regenerated up to 3 times (RFC 7217 §6).
- At link up `net` sends up to 3 router solicitations 4 s apart. A prefix information option with the autonomous flag and a /64 prefix yields an address; lifetimes follow RFC 4862 §5.5.3. Router lifetimes make the default routes; RDNSS options make learned DNS servers.

A static address is probed as DHCP and SLAAC addresses are; a conflict gives the interface `failed(net.duplicate-address)`.

## DNS

- Servers are tried in order: `NetConfig.dns.servers`, then, when `NetConfig.dns.learned` holds, those of DHCP option 6 and RDNSS options in arrival order.
- A resolution queries A and AAAA together over UDP port 53 from a random source port with a random id (RFC 5452), and again over TCP when the answer is truncated. A response counts only when its source address and port, id and question match the query.
- Each attempt waits 1 s; each server gets 2 attempts. When no server answers, the resolution fails with `net.unresolved`.
- The cache holds at most 1,024 names, positive answers for their TTL and at most 1 h, negative answers for the SOA minimum and at most 5 min (RFC 2308). It is dropped under memory [pressure](05-memory.md#pressure).
- Answers are `Untrusted`. An answer in a loopback, link-local, unspecified or multicast range, or equal to an address of the machine, is discarded.
- Names are absolute: there are no search domains. There is no DNSSEC validation.
- `net` resolves only the names of `connect` and `open_udp` calls that an `Egress` destination allows ([egress gate](#egress-gate)); it offers no resolution operation of its own. Connection attempts to the resolved addresses follow RFC 8305 with a 250 ms attempt delay.

## Egress gate

The **egress gate** is the part of `net` that every transmitted packet passes. Packets of `net`'s own protocols (ARP, neighbour discovery, MLD, DHCP, DNS queries, ICMP errors and echo replies, TCP segments without data) are charged to `net`'s account, DNS queries to the account of the call that caused them. Every other packet belongs to a **flow**: a TCP connection, a vsock stream connection, or a UDP association with one peer. An outbound flow, TCP or UDP, is opened through an `Egress` capability; an inbound one through a `Listen` capability ([ingress](#ingress)).

An `Egress` capability ([capability kinds](09-authority.md#capability-kinds)), minted from an `Egress` grant ([objects and rights](09-authority.md#objects-and-rights)), allows TCP and UDP flows to one destination: a host pattern and a port range (requires [N27](22-nan.md#n27)). It attenuates to a narrower pattern; `sys::Net.only_host(h)` narrows it to the name `h`.

```ebnf
HOST_PATTERN = DNS_NAME | "*." DNS_NAME | ( IPV4 | IPV6 ) "/" NUMBER ;
```

`DNS_NAME`, `IPV4` and `IPV6` are those of [origin labels](02-identifiers.md#origin-labels), and `NUMBER` that of [identifiers](02-identifiers.md#identifiers). `*.example.org` matches every name ending in `.example.org`, and `192.0.2.7/32` the one address.

When a flow opens, the gate checks the call against the capability: a host given as a name matches only name patterns and is then resolved ([DNS](#dns)); a host given as an address matches only address patterns; the port must lie in the range. A failed check fails the call with `net.denied` and emits `net.egress-denied`.

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `Egress.connect` | `host: text`, `port: u16` | `Conn`, a TCP connection | `net.denied`, `net.unresolved`, `net.unreachable`, `net.refused`, `net.timeout`, `net.flow-limit` | `irreversible{emit}` |
| `Egress.open_udp` | `host: text`, `port: u16` | `Datagrams` | `net.denied`, `net.unresolved`, `net.unreachable`, `net.flow-limit` | `none` |
| `Conn.read` | `max: u32` | `Untrusted[Bytes]`, empty once the peer has closed | `net.reset`, `net.revoked` | `none` |
| `Conn.write` | `data: Bytes` | `u32`, the bytes accepted | `net.reset`, `net.revoked`, `auth.budget-spent`, `budget.exhausted` | `irreversible{emit}` |
| `Conn.close` | — | — | `net.reset` | `irreversible{emit}` |
| `Conn.abandon` | — | — | — | `irreversible{emit}` |
| `Datagrams.send` | `data: Bytes` (at most 65,507 bytes) | — | `net.too-large`, `net.unreachable`, `net.revoked`, `auth.budget-spent`, `budget.exhausted` | `irreversible{emit}` |
| `Datagrams.receive` | — | `Untrusted[Bytes]` from the peer | `net.revoked` | `none` |
| `Datagrams.close` | — | — | — | `none` |

- A TCP connect not established within 15 s of its first SYN, or by the caller's deadline, fails with `net.timeout`. Keepalives are off.
- An account holds at most 1,024 open flows; another open fails with `net.flow-limit`. A flow's buffers grow to at most 4 MiB each way.
- Each flow carries the account and session of the message that opened it ([accounting](08-scheduling.md#accounting)), and its bytes are charged to that account. Bytes sent count against the grant's daily budget ([caveats](09-authority.md#caveats)), whose exhaustion fails the operation with `auth.budget-spent`, and against the `egress_bytes` budget of the session, whose exhaustion fails it with `budget.exhausted` ([budgets](18-link.md#budgets)).
- While open, a flow is a `LiveFlow` record at `/live/net/flows/<id>`: `id: u64`; `account: text::String`, the moniker or session id naming its account; `session: Option[SessionId]`; `principal: Principal`; `transport: FlowTransport`, one of `Tcp`, `Udp` and `Vsock`; `local: text::String` and `peer: text::String`, each `<address>:<port>`, an IPv6 address in brackets (RFC 3986 §3.2.2) and a vsock address as `<cid>:<port>`; `name: Option[text::String]`, the name it was opened with; `origin: OriginLabel` ([origin labels](#origin-labels)); `sent_bytes: u64`; `received_bytes: u64`; `opened: WallTime`; `as_of: u64`. Its end emits `net.flow-closed` with the record's fields and `duration_ns: u64`.
- Revoking the capability a flow was opened through resets the flow at once, and its pending operations fail with `net.revoked` ([revocation](09-authority.md#revocation)). `freeze egress`, which revokes every `Egress` capability, resets every outbound flow ([emergency controls](19-oversight.md#emergency-controls)).
- In a [preview](14-changes.md#preview) fork, operations classed above `recorded` are recorded and not performed.
- There is no filter language: the destinations of `Egress` and the ports of `Listen` are the only rules.

`sys::Net` on NaN OS ([jobs](06-execution.md#jobs)) maps onto these operations: `connect` is `Egress.connect`, `listen` uses the job's `Listen` capability for that port, and `sys::net::NetError` takes `Refused` from `net.refused`, `Reset` from `net.reset` and `net.revoked`, `Unreachable` from `net.unreachable`, `net.unresolved` and `net.timeout`, `AddressInUse` from `net.port-in-use`, and `Denied` from every other failure.

## Ingress

A `Listen` capability, minted from a `Listen` grant, receives the TCP connections, vsock stream connections and UDP datagrams addressed to its port on every local address (requires [N27](22-nan.md#n27)).

| Operation | Arguments | Result | Failures | Reversibility |
| --- | --- | --- | --- | --- |
| `Listen.accept` | — | `Conn`, with the peer's address | `net.port-in-use`, `net.revoked`, `net.flow-limit` | `none` |
| `Listen.receive` | — | `Datagram { peer: SocketAddr, data: Untrusted[Bytes] }` | `net.port-in-use`, `net.revoked` | `none` |
| `Listen.reply` | `peer: SocketAddr`, `data: Bytes` (at most 65,507 bytes) | — | `net.too-large`, `net.revoked` | `irreversible{emit}` |

A `SocketAddr` is `{ address: text::String, port: u16 }`, the address in its RFC text form.

- The port opens at the capability's first operation and closes when the capability is revoked or no holder remains. Opening a port another capability holds open fails with `net.port-in-use`.
- At most 128 connections wait for `accept` per port; beyond that, SYN cookies answer.
- A TCP segment to a closed port gets a reset; a datagram to a closed UDP port gets an ICMP port unreachable. ICMP errors and echo replies are limited to 100 per second per interface.
- An inbound flow is charged to the account of the instance that accepted it, and its replies need no `Egress`. Each accepted connection opens a [cause](17-observability.md#causes).

## Origin labels

The [origin label](02-identifiers.md#origin-labels) of the data a flow delivers is `net.<host>`: the name the flow was opened with, else the peer's address, and `vsock-<cid>` for a vsock peer. Flow records and `net.flow-closed` carry it. Every byte `net` delivers is `Untrusted`, and the origin sets of what an instance holding `Egress` or `Listen` capabilities writes follow [origin sets](19-oversight.md#origin-sets).

## Provisional changes

A write of `/sys/config/net` activates `net`, which carries every TCP and vsock session ([provisional commits](14-changes.md#provisional-commits)). The step [cuts](14-changes.md#provisional-commits) exactly the open TCP sessions whose connection, under the configuration it writes, loses its interface, its local address, its route to the peer through the same interface and gateway, or the `Listen` grant of its port. The [preview](14-changes.md#preview) lists those sessions, and a plan holding such a step gets the default `--confirm-within` ([options](14-changes.md#options)). vsock and serial sessions are never cut.

When the provisional commit is reverted for lack of a `confirm`, `net` reconciles to the restored configuration and releases the DHCP leases it obtained under the reverted one (DHCPRELEASE).

```text
> 4 plan [write /sys/config/net.default.ipv4 Ipv4Mode.Static(address: "10.0.0.9/24", gateway: Some("10.0.0.1"))]
```

The preview of this step lists the session connected through `10.0.0.5` as cut and adds `--confirm-within 120`; after `apply`, the client connects to `10.0.0.9` and sends `confirm` with the commit number.

## Network transports of the link

The [link](18-link.md#transports) holds a `Listen` grant for port 7710, written at [genesis](13-tree.md#genesis), and accepts TCP and vsock connections on it. A vsock connection carries the link's bytes as they are, its sessions opening by challenge and signature ([connections and sessions](18-link.md#connections-and-sessions)). A TCP connection runs one Noise_IK handshake (The Noise Protocol Framework, revision 34) before any link byte:

- **Protocol name:** `Noise_IK_25519_ChaChaPoly_BLAKE2s`, the client as initiator and the machine as responder, with the prologue `nanos link 1` (12 ASCII bytes). The cryptography runs in constant time (requires [N25](22-nan.md#n25)).
- **Responder keys:** the link key ([keys on the machine](09-authority.md#keys-on-the-machine)), an X25519 static key and an Ed25519 identity key that `keys` holds. Their public halves are the `link_keys` field of `/live/security` ([/live](13-tree.md#live)); clients pin them.
- **Initiator key:** the X25519 form (RFC 7748 §4.1 birational map) of the Ed25519 public key of an enrolled principal or of a delegated credential ([delegation](09-authority.md#delegation)). On a claimed machine, a handshake whose initiator key matches none fails with `net.unknown-key`, and the connection closes without a second message.
- **Framing:** each Noise message is preceded by its length as a 2-byte big-endian integer and is at most 65,535 bytes.
- **Messages:** message 1 (`e, es, s, ss`) has an empty payload. Message 2 (`e, ee, se`) carries as payload the 64-byte `handshake` [signature](09-authority.md#signatures) by the identity key; its object, of kind `handshake` ([hashes](02-identifiers.md#hashes)), is the 32-byte handshake hash before that payload. The client closes the connection when the signature does not verify. Transport messages then carry the link's byte stream both ways ([framing](18-link.md#framing)).
- **Keys:** `link` passes message 1 to `keys` through its `KeyUse` capability of purpose `handshake`, and `keys` returns message 2, the initiator's static public key, the handshake hash and the two transport keys; no private key leaves `keys`.
- **Binding:** the connection's binding derives from the final handshake hash ([connections and sessions](18-link.md#connections-and-sessions)); the principal whose key the initiator used is authenticated by the handshake alone.
- **Limits:** message 1 must arrive within 10 s of the connection opening. The machine runs at most 100 handshakes per second, 10 per peer address; a connection beyond them closes before its message 1 is processed. A transport message that fails to decrypt closes the connection. Each failure emits `net.handshake-failed`.

## Events

| Kind | Payload | Section |
| --- | --- | --- |
| `net.link-up`, `net.link-down` | interface, `LinkState` | [The network stack](#the-network-stack) |
| `net.address-added`, `net.address-removed` | interface, address, origin | [Address configuration](#address-configuration) |
| `net.address-conflict` | interface, address, the other host's link address | [Address configuration](#address-configuration) |
| `net.egress-denied` | account, session, principal, host, port | [Egress gate](#egress-gate) |
| `net.flow-closed` | the flow record, duration | [Egress gate](#egress-gate) |
| `net.handshake-failed` | peer address, reason (`timeout`, `unknown-key`, `limit`, `decrypt`) | [Network transports of the link](#network-transports-of-the-link) |

## Codes

| Code | Meaning | Section |
| --- | --- | --- |
| `net.denied` | the `Egress` capability's destination does not cover the host and port | [Egress gate](#egress-gate) |
| `net.unresolved` | no DNS server resolved the name | [DNS](#dns) |
| `net.unreachable` | no route leads to the peer, or neighbour discovery or ICMP reports it unreachable | [Egress gate](#egress-gate) |
| `net.refused` | the peer refused the connection | [Egress gate](#egress-gate) |
| `net.timeout` | the connection was not established in time | [Egress gate](#egress-gate) |
| `net.reset` | the peer reset the connection | [Egress gate](#egress-gate) |
| `net.revoked` | the capability the flow was opened through was revoked | [Egress gate](#egress-gate) |
| `net.flow-limit` | the account holds 1,024 open flows | [Egress gate](#egress-gate) |
| `net.too-large` | a datagram above 65,507 bytes | [Egress gate](#egress-gate) |
| `net.port-in-use` | another capability holds the port open | [Ingress](#ingress) |
| `net.duplicate-address` | another host uses the static address | [Address configuration](#address-configuration) |
| `net.no-lease` | no DHCP server answered 5 attempts | [Address configuration](#address-configuration) |
| `net.unknown-key` | the initiator's static key belongs to no enrolled principal or credential | [Network transports of the link](#network-transports-of-the-link) |
